```
способ 1 не помог, то же
docker build -t yugabyte-with-postgis-test:latest .
Sending build context to Docker daemon   25.6kB
Step 1/8 : FROM rockylinux:8 as postgis-builder
8: Pulling from library/rockylinux
9088cdb84e39: Retrying in 1 second
error pulling image configuration: download failed after attempts=6: x509: certificate signed by unknown authority

FROM centos:7 as postgis-builder 
так
docker build -t yugabyte-with-postgis-test:latest .
Sending build context to Docker daemon   25.6kB
Step 1/8 : FROM centos:7 as postgis-builder
7: Pulling from library/centos
2d473b07cdd5: Already exists
error pulling image configuration: download failed after attempts=6: x509: certificate signed by unknown authority

FROM ubuntu:20.04 as postgis-builder
так
docker build -t yugabyte-with-postgis-test:latest .
Sending build context to Docker daemon   25.6kB
Step 1/8 : FROM ubuntu:20.04 as postgis-builder
20.04: Pulling from library/ubuntu
13b7e930469f: Retrying in 1 second
error pulling image configuration: download failed after attempts=6: x509: certificate signed by unknown authority
```

Проблема явно в корневых сертификатах вашего Docker-хоста. Давайте сделаем две вещи: 1) проверим текущие сертификаты, 2) попробуем самый прямой путь - установить PostGIS **напрямую в ваш запущенный контейнер YugabyteDB**.

### 🛠️ Проверка и исправление сертификатов Docker

Выполните на хост-машине (не в контейнере):

```bash
# 1. Проверьте, какие сертификаты есть у Docker
docker run --rm -it alpine sh -c 'apk add --no-cache ca-certificates && ls -la /etc/ssl/certs/ | head -5'

# 2. Попробуйте явно указать Docker использовать системные сертификаты
# Создайте или отредактируйте /etc/docker/daemon.json
sudo tee /etc/docker/daemon.json <<-'EOF'
{
    "tls": true,
    "tlscacert": "/etc/ssl/certs/ca-certificates.crt",
    "tlscert": "/var/lib/docker/certs/client-cert.pem",
    "tlskey": "/var/lib/docker/certs/client-key.pem"
}
EOF

# 3. Перезапустите Docker
sudo systemctl restart docker

# 4. Проверьте скачивание простого образа
docker pull hello-world
```

### 🎯 Прямой путь: установка PostGIS в запущенный контейнер

Пока разбираемся с сертификатами, давайте установим PostGIS напрямую в ваш текущий контейнер YugabyteDB. Это временное решение, но оно позволит проверить работоспособность.

```bash
# 1. Зайдите в контейнер yb-tserver (из вашего docker-compose)
docker exec -it yugabyte_yb-tserver1_1 bash

# 2. Внутри контейнера выполните:
# Обновляем систему и устанавливаем ca-certificates (если их нет)
yum update -y
yum install -y ca-certificates

# 3. Устанавливаем PostGIS напрямую (вам нужно определить версию PostgreSQL в YugabyteDB)
# Сначала проверьте версию PostgreSQL:
/home/yugabyte/postgres/bin/postgres --version
# Пример вывода: postgres (PostgreSQL) 11.2-YB-2.19.2.0-b0

# 4. Исходя из версии (например, 11.2), устанавливаем PostGIS 3.1 для PostgreSQL 11
# Добавляем репозиторий PostgreSQL
rpm -Uvh https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Устанавливаем PostGIS
yum install -y postgis31_11 postgis31_11-client

# 5. Копируем файлы PostGIS в каталоги YugabyteDB
cp -r /usr/pgsql-11/lib/postgis* /home/yugabyte/postgres/lib/
cp -r /usr/pgsql-11/share/extension/postgis* /home/yugabyte/postgres/share/extension/

# 6. Выполняем перелинковку библиотек
/home/yugabyte/bin/post_install.sh -e

# 7. Выходим из контейнера
exit

# 8. Проверяем работу PostGIS
docker exec -it yugabyte_yb-tserver1_1 /home/yugabyte/bin/ysqlsh -c "CREATE EXTENSION postgis; SELECT PostGIS_Full_Version();"
```

### 🔄 Альтернатива: Сборка без Docker Hub

Если оба подхода не сработают, давайте соберём образ полностью локально. Создайте файл `Dockerfile.local`:

```bash
cat > Dockerfile.local << 'EOF'
# Используем текущий образ YugabyteDB как базовый
FROM yugabytedb/yugabyte:2.19.2.0-b121

# Внутри контейнера устанавливаем PostGIS
RUN yum update -y && \
    yum install -y ca-certificates && \
    rpm -Uvh https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm && \
    yum install -y postgis31_11 postgis31_11-client && \
    cp -r /usr/pgsql-11/lib/postgis* /home/yugabyte/postgres/lib/ && \
    cp -r /usr/pgsql-11/share/extension/postgis* /home/yugabyte/postgres/share/extension/ && \
    /home/yugabyte/bin/post_install.sh -e
EOF

# Соберите образ
docker build -f Dockerfile.local -t yugabyte-with-postgis-local:latest .
```

### 📊 Краткий план действий

1.  **Попробуйте установить PostGIS напрямую в контейнер** (пункт 2) - это самый быстрый способ проверки.
2.  **Если сработает** - у вас будет работающий PostGIS, и можно будет позже перенести эту логику в Dockerfile.
3.  **Если не сработает из-за проблем с репозиториями** - используйте `Dockerfile.local`, который не требует скачивания дополнительных базовых образов.
4.  **Для финального решения** нужно разобраться с корневыми сертификатами на хосте (обратитесь к системному администратору, если это корпоративная среда).

Начните с прямого пути - установки в запущенный контейнер. Каков результат?
