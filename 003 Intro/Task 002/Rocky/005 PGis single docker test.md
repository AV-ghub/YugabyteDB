```
да, давай начнем со стратегии быстрой проверки
но опять те же проблемы внутри контейнера
[]# yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
AlmaLinux 8 - BaseOS                                                                                                                                                          0.0  B/s |   0  B     00:04
Errors during downloading metadata for repository 'baseos':
  - Curl error (60): Peer certificate cannot be authenticated with given CA certificates for https://mirrors.almalinux.org/mirrorlist/8/baseos?countme=2 [SSL certificate problem: self signed certificate in certificate chain]
  - Curl error (60): Peer certificate cannot be authenticated with given CA certificates for https://mirrors.almalinux.org/mirrorlist/8/baseos [SSL certificate problem: self signed certificate in certificate chain]
Error: Failed to download metadata for repo 'baseos': Cannot prepare internal mirrorlist: Curl error (60): Peer certificate cannot be authenticated with given CA certificates for https://mirrors.almalinux.org/mirrorlist/8/baseos [SSL certificate problem: self signed certificate in certificate chain]
[]#
[]#
[]#
[]#
[]# yum update
AlmaLinux 8 - BaseOS                                                                                                                                                          0.0  B/s |   0  B     00:04
Errors during downloading metadata for repository 'baseos':
  - Curl error (60): Peer certificate cannot be authenticated with given CA certificates for https://mirrors.almalinux.org/mirrorlist/8/baseos [SSL certificate problem: self signed certificate in certificate chain]
Error: Failed to download metadata for repo 'baseos': Cannot prepare internal mirrorlist: Curl error (60): Peer certificate cannot be authenticated with given CA certificates for https://mirrors.almalinux.org/mirrorlist/8/baseos [SSL certificate problem: self signed certificate in certificate chain]
```

Ошибка связана с **системным временем внутри контейнера или с криптографическими политиками AlmaLinux**. Проблема не уникальна — она известна в сообществе AlmaLinux, и есть проверенные решения. Давайте диагностируем и устраним её напрямую.

### 🛠️ Диагностика и решение внутри контейнера
Войдите в контейнер и выполните проверки:

```bash
# Войдите в контейнер YugabyteDB
docker exec -it yugabyte bash

# Шаг 1: Проверьте системное время внутри контейнера
timedatectl status
# Посмотрите на строку "Local time" и "Universal time". Если время сильно расходится с реальным, это частая причина ошибки SSL.
# Пример правильного вывода:
#       Local time: Wed 2026-02-05 09:45:00 UTC
#       Universal time: Wed 2026-02-05 09:45:00 UTC

# Шаг 2: Проверьте текущую криптографическую политику
update-crypto-policies --show
# По умолчанию в AlmaLinux 8/9 установлена политика `DEFAULT`. Если там `FUTURE` или `LEGACY`, это может блокировать соединение с зеркалами.
```

### 🔧 Быстрые решения на выбор
В зависимости от результата диагностики выберите одно из решений **внутри контейнера**:

**Вариант A: Если время неверное**
Неправильное системное время — распространённая причина ошибки `certificate is not yet valid`.

```bash
# Временно отключите синхронизацию времени
timedatectl set-ntp false
# Установите точное время (пример для 5 февраля 2026, 9:45 UTC)
timedatectl set-time "2026-02-05 09:45:00"
# Включите синхронизацию обратно
timedatectl set-ntp true
# Повторите попытку обновить репозитории
yum update
```

**Вариант B: Если криптографическая политика не `DEFAULT`**
Политика `FUTURE` (используется в некоторых безопасных профилях) может быть слишком строгой для старых сертификатов репозиториев.

```bash
# Установите политику по умолчанию
update-crypto-policies --set DEFAULT
# Обновите доверенные сертификаты (это может занять минуту)
update-ca-trust
# Попробуйте установить пакеты снова
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

**Вариант C: Комплексное решение (если не помогли A и B)**
Выполните команды последовательно:

```bash
# 1. Сбросьте время и политики
timedatectl set-ntp false
timedatectl set-time "2026-02-05 09:45:00"
update-crypto-policies --set DEFAULT
update-ca-trust

# 2. Добавьте репозиторий PostgreSQL, временно игнорируя SSL-ошибки
# Используйте curl с флагом -k для пропуска проверки сертификата
curl -k -o /tmp/pgdg.rpm https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
rpm -ivh /tmp/pgdg.rpm

# 3. Установите PostGIS для PostgreSQL 15
yum install -y postgresql15-server postgis35_15
```

### 💡 Рекомендации и дальнейшие шаги
1.  **Начните с диагностики**: Выполните `timedatectl status` и `update-crypto-policies --show`. Это подскажет, какое решение применить.
2.  **Работайте внутри контейнера**: Все команды нужно выполнять в оболочке контейнера (`docker exec -it yugabyte bash`).
3.  **Если сработает**: После установки PostGIS вернитесь к шагам копирования файлов и выполнения `post_install.sh -e`, как было в предыдущем плане.

Скорее всего, проблема решится **Вариантом B** (сброс криптополитики), так как образы на базе AlmaLinux часто используют строгий профиль безопасности.

> **⚠️ Примечание по безопасности**: Установка криптополитики в `DEFAULT` безопасна для тестовой среды, но в продакшене следует согласовать этот параметр с политиками безопасности вашей организации.

Попробуйте эти шаги внутри контейнера `yugabyte` и сообщите, какой вариант сработал.
