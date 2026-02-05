```
нет, все плохо, возвращаемся к той же проблеме
docker build -f Dockerfile.build -t yugabyte-postgis-built:2025.2 .
[+] Building 5.3s (5/14)                                                                                                                                                                       docker:default
 => [internal] load build definition from Dockerfile.build                                                                                                                                               0.0s
 => => transferring dockerfile: 2.34kB                                                                                                                                                                   0.0s
 => [internal] load metadata for docker.io/yugabytedb/yugabyte:2025.2.0.1-b1                                                                                                                             0.0s
 => [internal] load .dockerignore                                                                                                                                                                        0.0s
 => => transferring context: 2B                                                                                                                                                                          0.0s
 => CACHED [ 1/11] FROM docker.io/yugabytedb/yugabyte:2025.2.0.1-b1                                                                                                                                      0.0s
 => ERROR [ 2/11] RUN yum install -y epel-release &&     yum install -y wget gcc make cmake git     postgresql15-devel     proj-devel proj-epsg proj-nad     geos-devel     gdal-devel     json-c-devel  5.2s
------
 > [ 2/11] RUN yum install -y epel-release &&     yum install -y wget gcc make cmake git     postgresql15-devel     proj-devel proj-epsg proj-nad     geos-devel     gdal-devel     json-c-devel     libxml2-devel     protobuf-c-devel     libtool:
5.146 AlmaLinux 8 - BaseOS                            0.0  B/s |   0  B     00:04
5.146 Errors during downloading metadata for repository 'baseos':
5.146   - Curl error (60): Peer certificate cannot be authenticated with given CA certificates for https://mirrors.almalinux.org/mirrorlist/8/baseos [SSL certificate problem: self signed certificate in certificate chain]
5.155 Error: Failed to download metadata for repo 'baseos': Cannot prepare internal mirrorlist: Curl error (60): Peer certificate cannot be authenticated with given CA certificates for https://mirrors.almalinux.org/mirrorlist/8/baseos [SSL certificate problem: self signed certificate in certificate chain]
------
Dockerfile.build:5
--------------------
   4 |     # 1. Устанавливаем ВСЕ необходимые зависимости для сборки
   5 | >>> RUN yum install -y epel-release && \
   6 | >>>     yum install -y wget gcc make cmake git \
   7 | >>>     postgresql15-devel \
   8 | >>>     proj-devel proj-epsg proj-nad \
   9 | >>>     geos-devel \
  10 | >>>     gdal-devel \
  11 | >>>     json-c-devel \
  12 | >>>     libxml2-devel \
  13 | >>>     protobuf-c-devel \
  14 | >>>     libtool
  15 |
--------------------
ERROR: failed to build: failed to solve: process "/bin/sh -c yum install -y epel-release &&     yum install -y wget gcc make cmake git     postgresql15-devel     proj-devel proj-epsg proj-nad     geos-devel     gdal-devel     json-c-devel     libxml2-devel     protobuf-c-devel     libtool" did not complete successfully: exit code: 1

похоже это бесперспективно, отнимет уйму времени и неизвестно чем закончится
а вариант отсюда https://dev.to/yugabyte/postgis-on-yugabytedb-alma8-with-workaround-2okj не операбельный?
или может быть просто поставить это https://github.com/giovannicandido/yugabytedb-postgis вариант?
что с того что он старый
нас же в принципе интересует функциональность постгиса по задаче
в первую очередь
и уже потом как мы будем это развертывать
что думаешь по этому поводу?
```

Вы абсолютно правы — сборка из исходников в текущих условиях (проблемы с репозиториями AlmaLinux) превратится в многочасовую битву. Давайте использовать **готовое решение**.

Оба предложенных вами варианта **работоспособны** и гораздо быстрее. Вот их сравнение:

| Критерий | **Готовый образ giovannicandido** | **Ручная установка по статье** |
|:----------|:-----------------------------------|:--------------------------------|
| **Сложность** | 🔴 **Очень низкая** (просто `docker pull`) | 🟡 **Средняя** (нужно выполнить несколько команд) |
| **Версия YugabyteDB** | ⚠️ **2.15.3** (старая, 2022 год) | ✅ **Любая** (включая вашу 2025.2) |
| **Версия PostGIS** | 3.1 (для PostgreSQL 11) | **Можно выбрать** (3.5 для pg15) |
| **Поддержка** | ❌ Не обновляется с 2022 | ✅ Сообщество, рабочая инструкция |
| **Совместимость** | ⚠️ Могут быть проблемы с новыми функциями | ✅ Полная совместимость с вашим кластером |

### 🚀 Рекомендация: **Использовать готовый образ для быстрого теста**
Поскольку главная цель — **проверить функциональность PostGIS**, начните с образа `giovannicandido/yugabytedb-postgis`. Это даст результат за 5 минут.

```bash
# 1. Скачайте и запустите готовый образ
docker run -d \
  --name yb-postgis-ready \
  -p 5441:5433 \
  -p 9046:9042 \
  -p 7006:7000 \
  giovannicandido/yugabytedb-postgis:latest

# 2. Дайте время на запуск (2-3 минуты)
sleep 180

# 3. Проверьте
docker exec yb-postgis-ready /home/yugabyte/bin/ysqlsh -h 172.17.0.2 -p 5433 -U yugabyte -c "CREATE EXTENSION postgis; SELECT PostGIS_Full_Version();"
```

**Важно:** В этом образе старая версия YugabyteDB (2.15.3). Если вам нужна **именно ваша версия 2025.2**, переходите ко второму способу.

### 🔧 Альтернатива: **Ручная установка по статье** (если нужна актуальная версия)
Если готовый образ не подходит по версии, выполните шаги из статьи. Вот адаптированная последовательность:

1. **Установите PostGIS в ваш текущий рабочий контейнер** (мы уже знаем, что он работает):
   ```bash
   # Войдите в контейнер
   docker exec -it yugabyte-pg bash
   
   # Внутри контейнера выполните (используем подход из статьи):
   # 1. Отключите SSL-проверку для yum
   echo "sslverify=0" >> /etc/yum.conf
   
   # 2. Установите репозитории и PostGIS для PostgreSQL 15
   yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
   yum install -y postgresql15-server postgis35_15 postgis35_15-client
   
   # 3. Скопируйте файлы PostGIS
   cp -r /usr/pgsql-15/lib/postgis* /home/yugabyte/postgres/lib/
   cp -r /usr/pgsql-15/share/extension/postgis* /home/yugabyte/postgres/share/extension/
   
   # 4. Перелинкуйте библиотеки
   /home/yugabyte/bin/post_install.sh -e
   ```

2. **Выйдите из контейнера и проверьте**:
   ```bash
   exit
   docker exec yugabyte-pg /home/yugabyte/bin/ysqlsh -h 172.16.0.2 -p 5433 -U yugabyte -c "CREATE EXTENSION postgis; SELECT PostGIS_Full_Version();"
   ```

### 💡 Что выбрать?
*   **Для быстрого теста функций PostGIS** → Готовый образ `giovannicandido`.
*   **Для интеграции PostGIS с вашим кластером 2025.2** → Ручная установка по статье.

**Попробуйте готовый образ.** Если он удовлетворит ваши потребности по функциональности — задача решена. Если нет — вернёмся к ручной установке, но уже с пониманием, что это рабочий путь.
