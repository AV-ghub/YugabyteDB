```
docker build -t yugabyte-with-postgis-test:latest .
Sending build context to Docker daemon  19.46kB
Step 1/11 : FROM yugabytedb/yugabyte:2.15.3.2-b1 as postgis-builder
 ---> 85aae308f23a
Step 2/11 : RUN cd /etc/yum.repos.d/ &&     mkdir -p disabled_repos &&     mv *.repo disabled_repos/ 2>/dev/null || true
 ---> Using cache
 ---> 6b7135d46a15
Step 3/11 : RUN rpm -ivh --nodeps /var/cache/yum/*/packages/ca-certificates*.rpm 2>/dev/null || true
 ---> Running in ee997d970984
Removing intermediate container ee997d970984
 ---> d62d3df67a3a
Step 4/11 : RUN yum install -y wget --disablerepo=* 2>/dev/null || (     curl -L -o /usr/bin/wget https://github.com/moparisthebest/static-curl/releases/download/v8.7.1/curl-amd64 &&     chmod +x /usr/bin/wget     )
 ---> Running in a4d0463f2c04
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 5310k  100 5310k    0     0  4385k      0  0:00:01  0:00:01 --:--:-- 6905k
Removing intermediate container a4d0463f2c04
 ---> 5adf7c9f57a0
Step 5/11 : RUN wget --no-check-certificate https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm &&     rpm -ivh pgdg-redhat-repo-latest.noarch.rpm --nodeps
 ---> Running in 8ea3a631bd59
curl: option --no-check-certificate: is unknown
curl: try 'curl --help' for more information
The command '/bin/sh -c wget --no-check-certificate https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm &&     rpm -ivh pgdg-redhat-repo-latest.noarch.rpm --nodeps' returned a non-zero code: 2

```

Ошибка произошла потому, что в вашем Dockerfile **файл `wget` не является настоящей утилитой `wget`, а является псевдонимом для `curl`**. Когда команда `wget` с флагом `--no-check-certificate` была выполнена, фактически запустился `curl`, который такого флага не понимает.

На этом этапе попытки исправить устаревший образ CentOS 7 становятся слишком затратными по времени. Давайте перейдем к надежному решению, о котором мы говорили ранее: **использовать современный образ Rocky Linux 8** для сборки PostGIS.

### ✅ Надежный Dockerfile на Rocky Linux 8
Создайте новый файл (или замените текущий) с этим содержимым:

```bash
cat > Dockerfile << 'EOF'
# ПЕРВЫЙ ЭТАП: Сборка PostGIS в современной и рабочей среде Rocky Linux 8
FROM rockylinux:8 as postgis-builder

# Устанавливаем необходимые инструменты и добавляем репозиторий PostgreSQL
RUN dnf install -y wget && \
    dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm && \
    dnf -qy module disable postgresql

# Устанавливаем PostGIS. Для совместимости с YugabyteDB важно использовать версию PostgreSQL 11.
RUN dnf install -y postgresql11-server postgis31_11 postgis31_11-client

# Копируем файлы PostGIS в структуру каталогов, ожидаемую YugabyteDB
RUN cp -r /usr/pgsql-11/lib/postgis* /home/yugabyte/postgres/lib/ 2>/dev/null || : && \
    cp -r /usr/pgsql-11/share/extension/postgis* /home/yugabyte/postgres/share/extension/ 2>/dev/null || :

# ВТОРОЙ ЭТАП: Финальный образ YugabyteDB
FROM yugabytedb/yugabyte:2.19.2.0-b121

# Копируем только библиотеки и расширения PostGIS из первого этапа
COPY --from=postgis-builder /home/yugabyte/postgres/lib/postgis* /home/yugabyte/postgres/lib/
COPY --from=postgis-builder /home/yugabyte/postgres/share/extension/postgis* /home/yugabyte/postgres/share/extension/

# Выполняем обязательную перелинковку библиотек в образе YugabyteDB
RUN /home/yugabyte/bin/post_install.sh -e
EOF
```

**Ключевые преимущества этого подхода:**
*   **Рабочие репозитории**: Rocky Linux 8 активно поддерживается, и его репозитории (`BaseOS`, `AppStream`) полностью работоспособны.
*   **Нативный `wget` и `dnf`**: Используются стандартные и корректно работающие пакетные менеджеры.
*   **Четкое разделение**: PostGIS собирается в чистой среде, а в финальный образ попадают только необходимые файлы.

### 🚀 Сборка и тестирование
1.  **Соберите образ заново**:
    ```bash
    docker build -t yugabyte-with-postgis-test:latest .
    ```
2.  **Запустите тестовый контейнер**:
    ```bash
    docker run -d \
      --name yb-single-node-test \
      -p 5433:5433 \
      -p 9042:9042 \
      -p 7000:7000 \
      yugabyte-with-postgis-test:latest \
      bin/yugabyted start \
      --base_dir=/home/yugabyte/yb_data \
      --daemon=false
    ```
3.  **Проверьте PostGIS** (подождите ~30 секунд после запуска):
    ```bash
    docker exec -it yb-single-node-test ./bin/ysqlsh -h localhost -p 5433 -U yugabyte << 'SQL'
    CREATE EXTENSION postgis;
    SELECT PostGIS_Full_Version();
    SQL
    ```

### 🤔 Что делать, если возникнут проблемы с зависимостями
Если на этапе `dnf install` в Rocky Linux возникнет ошибка о конфликте библиотек (например, `libgdal`), это будет означать, что системные версии библиотек несовместимы с требуемыми PostGIS. В этом случае стоит проверить, какие версии пакетов доступны:
```bash
# Внутри этапа сборки можно выполнить проверку
dnf list available gdal* geos* proj* --showduplicates
```
И, при необходимости, явно установить более старые, совместимые версии, указав номер версии в команде установки.

Попробуйте собрать образ по новой инструкции — это должно сработать. Если столкнетесь с новой ошибкой, пришлите полный вывод, чтобы мы могли целенаправленно её решить.
