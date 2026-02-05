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

-----------------------------------------------
-------------------------------------------------
--------------------------------------------

```
вот так теперь
[]# yum update
AlmaLinux 8 - BaseOS                                                                                                                                                          0.0  B/s |   0  B     00:04
Errors during downloading metadata for repository 'baseos':
  - Curl error (60): Peer certificate cannot be authenticated with given CA certificates for https://mirrors.almalinux.org/mirrorlist/8/baseos [SSL certificate problem: self signed certificate in certificate chain]
Error: Failed to download metadata for repo 'baseos': Cannot prepare internal mirrorlist: Curl error (60): Peer certificate cannot be authenticated with given CA certificates for https://mirrors.almalinux.org/mirrorlist/8/baseos [SSL certificate problem: self signed certificate in certificate chain]
[]#
[]#
[]# timedatectl status
System has not been booted with systemd as init system (PID 1). Can't operate.
Failed to create bus connection: Host is down
[]#
[]#
[]#
[]# update-crypto-policies --show
bash: update-crypto-policies: command not found
[]#
[]#
[]# update-ca-trust
[]#
[]#
[]# update-crypto-policies --set DEFAULT
bash: update-crypto-policies: command not found
[]#
[]#
[]# yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
AlmaLinux 8 - BaseOS                                                                                                                                                          0.0  B/s |   0  B     00:04
Errors during downloading metadata for repository 'baseos':
  - Curl error (60): Peer certificate cannot be authenticated with given CA certificates for https://mirrors.almalinux.org/mirrorlist/8/baseos [SSL certificate problem: self signed certificate in certificate chain]
Error: Failed to download metadata for repo 'baseos': Cannot prepare internal mirrorlist: Curl error (60): Peer certificate cannot be authenticated with given CA certificates for https://mirrors.almalinux.org/mirrorlist/8/baseos [SSL certificate problem: self signed certificate in certificate chain]
```

Вы не можете выполнить основные проверки (`timedatectl`, `update-crypto-policies`), потому что в базовом Docker-образе YugabyteDB, скорее всего, нет системы `systemd` и минимального набора системных утилит. Это стандартная практика для оптимизации размера контейнеров.

Основная проблема в том, что в контейнере отсутствуют или устарели корневые SSL-сертификаты (`ca-certificates`), которые нужны для проверки сертификата репозитория AlmaLinux. Без них `yum` не может доверять HTTPS-соединению.

Вот два рабочих обходных пути, начиная с самого простого.

### ⚡ Вариант 1: Быстрый обход — использование HTTP вместо HTTPS
Самый простой способ — заставить `yum` использовать протокол HTTP для зеркал, где это возможно, минуя проверку SSL.

Внутри контейнера выполните:
```bash
# 1. Отредактируйте конфигурацию репозитория baseos
sed -i 's|^mirrorlist=https|mirrorlist=http|g; s|^#baseurl=http|baseurl=http|g' /etc/yum.repos.d/almalinux*.repo

# 2. Очистите кэш
yum clean all

# 3. Теперь попробуйте установить пакеты для PostGIS
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

> ⚠️ **Примечание**: Использование HTTP снижает безопасность, но для быстрой установки пакетов внутри тестового контейнера это допустимо. После установки PostGIS это изменение можно вернуть обратно.

### 🔧 Вариант 2: Установка недостающих пакетов в обход `yum`
Если Вариант 1 не сработает, можно вручную обновить `ca-certificates`, скачав свежий пакет с репозитория через `curl`.

Выполните в контейнере следующую последовательность команд:
```bash
# 1. Скачайте свежий пакет ca-certificates с репозитория AlmaLinux.
# Используем опцию -k для игнорирования ошибок SSL на этом шаге.
curl -k -o /tmp/ca-certificates.rpm http://repo.almalinux.org/almalinux/8/BaseOS/x86_64/os/Packages/ca-certificates-2023.2.60-80.0.el8.noarch.rpm

# 2. Установите пакет вручную
rpm -Uvh /tmp/ca-certificates.rpm --nodeps

# 3. Обновите доверенные сертификаты в системе
update-ca-trust

# 4. Теперь попробуйте снова выполнить yum install для репозитория PostgreSQL
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

> **Как проверить, что это сработало?** После успешной установки `pgdg-redhat-repo` можно продолжить установку PostGIS, как было в нашем предыдущем плане:
> 1. `yum install -y postgresql15-server postgis35_15` (или другой версии для pg15).
> 2. Скопировать файлы PostGIS в каталоги YugabyteDB.
> 3. Выполнить `/home/yugabyte/bin/post_install.sh -e`.

### 💡 Итог и рекомендация
| Подход | Суть | Когда использовать |
| :--- | :--- | :--- |
| **Вариант 1** | Временно перевести репозиторий `baseos` на HTTP. | Самый быстрый способ для теста. |
| **Вариант 2** | Вручную обновить пакет `ca-certificates`. | Если Вариант 1 не сработает или нужен более надежный путь. |

**Рекомендую начать с Варианта 1.** Он проще и, вероятно, решит проблему.

Если ни один из способов не поможет или возникнут новые ошибки, напишите — будем разбираться дальше.



--------------------------------------------
----------------------------------------
-------------------------------------------


```
нет, все то же
[]# sed -i 's|^mirrorlist=https|mirrorlist=http|g; s|^#baseurl=http|baseurl=http|g' /etc/yum.repos.d/almalinux*.repo
[]# yum clean all
0 files removed
[]#
[]#
[]# yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
AlmaLinux 8 - BaseOS                                                                    [===                                                                                ] ---  B/s |   0  B     --:-- ETA
AlmaLinux 8 - BaseOS                                                                                                                                                          0.0  B/s |   0  B     00:05
Errors during downloading metadata for repository 'baseos':
  - Curl error (60): Peer certificate cannot be authenticated with given CA certificates for https://mirrors.almalinux.org:443/mirrorlist/8/baseos [SSL certificate problem: self signed certificate in certificate chain]
Error: Failed to download metadata for repo 'baseos': Cannot prepare internal mirrorlist: Curl error (60): Peer certificate cannot be authenticated with given CA certificates for https://mirrors.almalinux.org:443/mirrorlist/8/baseos [SSL certificate problem: self signed certificate in certificate chain]
[]#
[]# curl -k -o /tmp/ca-certificates.rpm http://repo.almalinux.org/almalinux/8/BaseOS/x86_64/os/Packages/ca-certificates-2023.2.60-80.0.el8.noarch.rpm
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
[]#
[]# rpm -Uvh /tmp/ca-certificates.rpm --nodeps
error: /tmp/ca-certificates.rpm: not an rpm package (or package manifest):
[]#
[]# update-ca-trust
[]#
[]# yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
AlmaLinux 8 - BaseOS                                                                                                                                                          0.0  B/s |   0  B     00:05
Errors during downloading metadata for repository 'baseos':
  - Curl error (60): Peer certificate cannot be authenticated with given CA certificates for https://mirrors.almalinux.org:443/mirrorlist/8/baseos [SSL certificate problem: self signed certificate in certificate chain]
Error: Failed to download metadata for repo 'baseos': Cannot prepare internal mirrorlist: Curl error (60): Peer certificate cannot be authenticated with given CA certificates for https://mirrors.almalinux.org:443/mirrorlist/8/baseos [SSL certificate problem: self signed certificate in certificate chain]
```

Проблема глубже, чем ожидалось: `curl` скачал пустой файл (0 байт), поэтому `rpm` не смог его установить. Видимо, в контейнере нет нормальной сети или репозитории AlmaLinux недоступны.

Нам нужно **полностью отказаться от установки через yum внутри контейнера** и использовать более прямой подход: скопировать файлы PostGIS **сразу из официального образа `postgis/postgis`** в наш контейнер YugabyteDB.

Этот метод использует механизм **мультистейдж-сборки (multi-stage build)** Docker: мы временно запускаем контейнер с PostGIS, берем оттуда нужные файлы и копируем их в финальный образ.

### 🛠️ Окончательное решение: Мультистейдж-сборка с копированием PostGIS

Создайте новый `Dockerfile` (например, `Dockerfile.postgis.final`) на вашей хост-машине:

```dockerfile
# Этап 1: Берем официальный образ PostGIS, который собран для PostgreSQL 15
FROM postgis/postgis:15-3.5 AS postgis_source

# Этап 2: Берем работающий образ YugabyteDB, с которым у вас уже всё получилось
FROM yugabytedb/yugabyte:2025.2.0.1-b1

# Ключевой шаг: копируем ВСЕ файлы расширения PostGIS из первого контейнера
# Мы копируем библиотеки (.so файлы) и SQL-скрипты расширений (.sql, .control)
COPY --from=postgis_source /usr/local/lib/postgresql/postgis*.so /home/yugabyte/postgres/lib/
COPY --from=postgis_source /usr/local/share/postgresql/extension/postgis* /home/yugabyte/postgres/share/extension/

# Обязательно перелинковываем библиотеки в образе YugabyteDB
RUN /home/yugabyte/bin/post_install.sh -e

# Опционально: можно сразу создать расширение при запуске контейнера
# CMD ["/home/yugabyte/bin/yugabyted", "start", "--background=false", "--ysql_enable_auth=false"]
```

**Почему это должно сработать:**
*   Мы используем официальный, гарантированно рабочий образ `postgis/postgis`.
*   Мы не пытаемся что-то скачивать или устанавливать внутри проблемного контейнера.
*   Мы копируем только конечные файлы библиотек и расширений.

### 🚀 Сборка и запуск нового образа

1.  **Соберите новый образ** на хост-машине:
    ```bash
    docker build -f Dockerfile.postgis.final -t yugabyte-with-postgis-final:2025.2 .
    ```

2.  **Запустите контейнер из нового образа**, освободив порты (например, `5440`):
    ```bash
    docker run -d \
      --name yb-postgis-final \
      -p 5440:5433 \
      -p 9045:9042 \
      -p 7005:7000 \
      yugabyte-with-postgis-final:2025.2 \
      /home/yugabyte/bin/yugabyted start --background=false
    ```

3.  **Дайте контейнеру время на запуск** (1-2 минуты), затем проверьте:
    ```bash
    # Подключитесь и создайте расширение
    docker exec yb-postgis-final /home/yugabyte/bin/ysqlsh -h 172.17.0.2 -p 5433 -U yugabyte -c "CREATE EXTENSION postgis; SELECT PostGIS_Full_Version();"
    ```

### 🔍 Если при копировании возникнут ошибки (файлы не найдены)
Пакеты в образе `postgis/postgis` могут лежать в других директориях. Для диагностики можно временно запустить контейнер-источник и посмотреть, что в нем:

```bash
# Запустите временный контейнер для изучения структуры
docker run --rm -it postgis/postgis:15-3.5 bash -c "find /usr -name '*postgis*' 2>/dev/null | head -20"

# Вывод покажет правильные пути, например:
# /usr/local/lib/postgresql/postgis-3.so
# /usr/local/share/postgresql/extension/postgis.control
# ...
```

Если пути отличаются, просто обновите пути `COPY` в `Dockerfile` на те, которые вы увидели в выводе команды `find`.

### 💎 Резюме и следующий шаг
Проблема сети/репозиториев внутри минималистичного контейнера YugabyteDB оказалась тупиковой. Предложенный метод с мультистейдж1-сборкой — это стандартный и надежный способ добавления функциональности в Docker-образ, когда установка пакетов невозможна.

**Ваш следующий шаг:** Создайте и соберите `Dockerfile.postgis.final` по инструкции выше. Это самый прямой путь к цели.

