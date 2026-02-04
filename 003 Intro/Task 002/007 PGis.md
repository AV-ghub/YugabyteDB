```
docker build -t yugabyte-with-postgis-test:latest .
Sending build context to Docker daemon  15.87kB
Step 1/10 : FROM yugabytedb/yugabyte:2.15.3.2-b1 as postgis-builder
 ---> 85aae308f23a
Step 2/10 : RUN cd /etc/yum.repos.d/ &&     mkdir -p disabled_repos &&     mv *.repo disabled_repos/ 2>/dev/null || true
 ---> Running in 33179560edbe
Removing intermediate container 33179560edbe
 ---> 6b7135d46a15
Step 3/10 : RUN yum install -y wget --disablerepo=* 2>/dev/null || (     curl -L -o /usr/bin/wget https://github.com/moparisthebest/static-curl/releases/download/v8.7.1/curl-amd64 &&     chmod +x /usr/bin/wget     )
 ---> Running in dbaffb95509b
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 5310k  100 5310k    0     0  3640k      0  0:00:01  0:00:01 --:--:-- 7944k
Removing intermediate container dbaffb95509b
 ---> 6290eaa95036
Step 4/10 : RUN wget https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm &&     rpm -ivh pgdg-redhat-repo-latest.noarch.rpm --nodeps
 ---> Running in bfa70fd91e05
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
curl: (77) error setting certificate file: /etc/ssl/certs/ca-certificates.crt
The command '/bin/sh -c wget https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm &&     rpm -ivh pgdg-redhat-repo-latest.noarch.rpm --nodeps' returned a non-zero code: 77

```

Отлично, прогресс есть! Теперь проблема в SSL-сертификатах — `wget` (который фактически использует `curl`) не может проверить сертификат из-за отсутствия пакета `ca-certificates`. Это легко исправить.

### ✅ Исправленный Dockerfile

Нужно установить `ca-certificates` ДО того, как мы будем что-либо скачивать по HTTPS. Поскольку мы отключили репозитории, придётся установить его либо из локального кэша (если есть), либо ручным способом.

Замените содержимое `Dockerfile`:

```bash
cat > Dockerfile << 'EOF'
# Первый этап: Установка PostGIS в среде, совместимой с YugabyteDB
FROM yugabytedb/yugabyte:2.15.3.2-b1 as postgis-builder

# 1. Отключаем репозитории, чтобы yum не пытался к ним обращаться
RUN cd /etc/yum.repos.d/ && \
    mkdir -p disabled_repos && \
    mv *.repo disabled_repos/ 2>/dev/null || true

# 2. Устанавливаем ca-certificates ИЗ ЛОКАЛЬНОГО КЭША (если есть)
#    Если в кэше нет, используем обходной путь с --nodeps
RUN rpm -ivh --nodeps /var/cache/yum/*/packages/ca-certificates*.rpm 2>/dev/null || true

# 3. Устанавливаем wget (теперь с сертификатами должно работать)
RUN yum install -y wget --disablerepo=* 2>/dev/null || ( \
    # Если yum не может найти wget в кэше, используем curl
    curl -L -o /usr/bin/wget https://github.com/moparisthebest/static-curl/releases/download/v8.7.1/curl-amd64 && \
    chmod +x /usr/bin/wget \
    )

# 4. Скачиваем и устанавливаем репозиторий PostgreSQL (PGDG)
RUN wget --no-check-certificate https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm && \
    rpm -ivh pgdg-redhat-repo-latest.noarch.rpm --nodeps

# 5. Устанавливаем PostGIS 3.1 для PostgreSQL 11 ИСКЛЮЧИТЕЛЬНО из репозитория PGDG
RUN yum --disablerepo=* --enablerepo=pgdg-common --enablerepo=pgdg11 install -y \
    postgresql11-server \
    postgis31_11 \
    postgis31_11-client

# 6. Копируем файлы PostGIS в стандартные каталоги YugabyteDB
RUN cp -r /usr/pgsql-11/lib/postgis* /home/yugabyte/postgres/lib/ && \
    cp -r /usr/pgsql-11/share/extension/postgis* /home/yugabyte/postgres/share/extension/

# Второй этап: Финальный образ
FROM yugabytedb/yugabyte:2.19.2.0-b121

# Копируем только нужные файлы из первого этапа
COPY --from=postgis-builder /home/yugabyte/postgres/lib/postgis* /home/yugabyte/postgres/lib/
COPY --from=postgis-builder /home/yugabyte/postgres/share/extension/postgis* /home/yugabyte/postgres/share/extension/

# Выполняем перелинковку библиотек
RUN /home/yugabyte/bin/post_install.sh -e
EOF
```

### 🔧 Что изменилось:

1.  **Добавлена установка `ca-certificates`** (строки 12-13): Пытаемся установить пакет из локального кэша RPM. Если его нет — пропускаем ошибку (`2>/dev/null || true`).
2.  **Добавлен флаг `--no-check-certificate` к `wget`** (строка 22): На случай, если установка `ca-certificates` не удалась, этот флаг отключает проверку SSL-сертификатов. Это безопасно для скачивания пакетов с официального сайта PostgreSQL.

### 🚀 Альтернатива: Использование `rockylinux:8`

Если этот вариант всё равно не сработает, **самый надёжный путь** — использовать свежий образ `rockylinux:8` для сборки PostGIS. Это полностью решит проблемы с репозиториями и SSL.

Замените **только первую строку** Dockerfile на:
```dockerfile
FROM rockylinux:8 as postgis-builder
```

После этого **удалите шаги, связанные с отключением репозиториев и установкой `ca-certificates`** (строки 6-13 в текущем Dockerfile), так как в Rocky Linux 8 репозитории работают из коробки.

### 📝 Краткий план действий:

1.  **Попробуйте собрать образ** с текущим исправленным Dockerfile.
2.  **Если будет ошибка** — переключитесь на `FROM rockylinux:8` и упростите Dockerfile.
3.  **В случае успешной сборки** протестируйте PostGIS:

```bash
# Запустите одноузловой контейнер
docker run -d \
  --name yb-test \
  -p 5433:5433 \
  -p 9042:9042 \
  -p 7000:7000 \
  yugabyte-with-postgis-test:latest \
  bin/yugabyted start --daemon=false

# Проверьте PostGIS
sleep 30  # дайте время на запуск
docker exec yb-test ./bin/ysqlsh -c "CREATE EXTENSION postgis; SELECT PostGIS_Full_Version();"
```

Пробуйте и сообщайте о результате. Скорее всего, переход на `rockylinux:8` будет самым быстрым решением.
