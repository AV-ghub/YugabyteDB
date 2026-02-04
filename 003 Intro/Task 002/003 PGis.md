```
да, начал собирать, в итоге так
docker build -t yugabyte-with-postgis-test:latest .
Sending build context to Docker daemon  3.072kB
Step 1/7 : FROM yugabytedb/yugabyte:2.15.3.2-b1 as postgis-builder
2.15.3.2-b1: Pulling from yugabytedb/yugabyte
2d473b07cdd5: Pull complete
1b304b5eca1b: Pull complete
f731f709bf52: Pull complete
13fd6272489c: Pull complete
58b7eb6d39ab: Pull complete
2ee801689a5d: Pull complete
b2795496edea: Pull complete
7521f457f7e1: Pull complete
fea94e864fad: Pull complete
be71981095e7: Pull complete
ca9968672afe: Pull complete
74d73f907fc3: Pull complete
1e1a0dcd9420: Pull complete
d67ffa9f85e8: Pull complete
e104ebf5a18c: Pull complete
Digest: sha256:6d91e01eef912fbe2013400c4627a1967900c4dd578f800ed502d4e77bee84d4
Status: Downloaded newer image for yugabytedb/yugabyte:2.15.3.2-b1
 ---> 85aae308f23a
Step 2/7 : RUN yum update -y &&     yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm &&     yum install -y postgresql11-server postgis31_11 postgis31_11-client
 ---> Running in d39e06863597
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
Could not retrieve mirrorlist http://mirrorlist.centos.org/?release=7&arch=x86_64&repo=os&infra=container error was
14: curl#6 - "Could not resolve host: mirrorlist.centos.org; Unknown error"


 One of the configured repositories failed (Unknown),
 and yum doesn't have enough cached data to continue. At this point the only
 safe thing yum can do is fail. There are a few ways to work "fix" this:

     1. Contact the upstream for the repository and get them to fix the problem.

     2. Reconfigure the baseurl/etc. for the repository, to point to a working
        upstream. This is most often useful if you are using a newer
        distribution release than is supported by the repository (and the
        packages for the previous distribution release still work).

     3. Run the command with the repository temporarily disabled
            yum --disablerepo=<repoid> ...

     4. Disable the repository permanently, so yum won't use it by default. Yum
        will then just ignore the repository until you permanently enable it
        again or use --enablerepo for temporary usage:

            yum-config-manager --disable <repoid>
        or
            subscription-manager repos --disable=<repoid>

     5. Configure the failing repository to be skipped, if it is unavailable.
        Note that yum will try to contact the repo. when it runs most commands,
        so will have to try and fail each time (and thus. yum will be be much
        slower). If it is a very temporary problem though, this is often a nice
        compromise:

            yum-config-manager --save --setopt=<repoid>.skip_if_unavailable=true

Cannot find a valid baseurl for repo: base/7/x86_64
The command '/bin/sh -c yum update -y &&     yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm &&     yum install -y postgresql11-server postgis31_11 postgis31_11-client' returned a non-zero code: 1
```


У вас возникла сетевая ошибка при сборке Docker-образа. Проблема в том, что базовый образ `yugabytedb/yugabyte:2.15.3.2-b1` (основанный на CentOS 7) не может обратиться к репозиторию CentOS через `yum` из-за нерабочего зеркала `mirrorlist.centos.org`.

Это не ошибка в вашем Dockerfile, а старая известная проблема с CentOS 7. Давайте починим Dockerfile, добавив исправление для сети.

### 🔧 Исправленный Dockerfile
Создайте новый файл Dockerfile, в котором **перед обновлением системы** мы исправим конфигурацию репозиториев, заменив нерабочий `mirrorlist` на рабочий `baseurl`.

Скопируйте и выполните эту команду в вашей директории для сборки (`~/yugabyte-postgis-test`):

```bash
cat > Dockerfile << 'EOF'
# Первый этап: Установка PostGIS в среде, совместимой с YugabyteDB
FROM yugabytedb/yugabyte:2.15.3.2-b1 as postgis-builder

# Исправление сети CentOS 7: заменяем mirrorlist на vault.centos.org
RUN sed -i 's|^mirrorlist=|#mirrorlist=|g' /etc/yum.repos.d/CentOS-*.repo && \
    sed -i 's|^#baseurl=http://mirror.centos.org/centos|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*.repo

RUN yum update -y && \
    yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm && \
    yum install -y postgresql11-server postgis31_11 postgis31_11-client

# Копируем файлы PostGIS в стандартные каталоги YugabyteDB
RUN cp -r /usr/pgsql-11/lib/postgis* /home/yugabyte/postgres/lib/ && \
    cp -r /usr/pgsql-11/share/extension/postgis* /home/yugabyte/postgres/share/extension/

# Второй этап: Финальный образ (можно выбрать актуальную версию)
FROM yugabytedb/yugabyte:2.19.2.0-b121

# Копируем только нужные файлы из первого этапа
COPY --from=postgis-builder /home/yugabyte/postgres/lib/postgis* /home/yugabyte/postgres/lib/
COPY --from=postgis-builder /home/yugabyte/postgres/share/extension/postgis* /home/yugabyte/postgres/share/extension/

# Выполняем перелинковку библиотек (ключевой шаг!)
RUN /home/yugabyte/bin/post_install.sh -e
EOF
```

**Ключевое изменение:** Две строки с `sed` заменяют ссылки на нерабочие зеркала в репозиториях CentOS 7 на архивный сервер `vault.centos.org`, который доступен.

### 🚀 Сборка и тест образа
Теперь попробуйте собрать образ снова. Исправление должно решить проблему с сетью.

```bash
# 1. Соберите исправленный образ
docker build -t yugabyte-with-postgis-test:latest .

# 2. Запустите одноузловой контейнер для теста
docker run -d \
  --name yb-single-node-test \
  -p 5433:5433 \
  -p 9042:9042 \
  -p 7000:7000 \
  yugabyte-with-postgis-test:latest \
  bin/yugabyted start \
  --base_dir=/home/yugabyte/yb_data \
  --daemon=false

# 3. Дайте кластеру время на запуск (30-60 секунд)
sleep 60

# 4. Проверьте работу PostGIS
docker exec -it yb-single-node-test ./bin/ysqlsh -h localhost -p 5433 -U yugabyte << 'SQL'
CREATE EXTENSION postgis;
SELECT PostGIS_Full_Version();
SQL
```

### 🤔 Если сборка все равно не проходит
Если на этапе `yum install` появляются новые ошибки (например, о конфликтах версий библиотек `geos` или `gdal`), это известная проблема при обновлении PostGIS, связанная с зависимостями. В таком случае поможет более простой и надежный подход.

#### Альтернативный подход: Копирование файлов из готового образа PostGIS
Вместо сборки PostGIS внутри контейнера можно скопировать уже готовые файлы из официального образа `postgis/postgis`, используя ту же версию PostgreSQL, что и в YugabyteDB (например, 11.2). Этот метод часто используется сообществом.

```bash
# Создайте новый Dockerfile.alternative
cat > Dockerfile.alternative << 'EOF'
FROM yugabytedb/yugabyte:2.19.2.0-b121

# Временно запускаем контейнер PostGIS с нужной версией PostgreSQL (11)
# и копируем из него файлы расширения
RUN docker run --rm -v /tmp/postgis-files:/to-host postgis/postgis:11-3.1 \
    bash -c "cp -r /usr/local/lib/postgresql/postgis* /to-host/lib/ 2>/dev/null || true && \
             cp -r /usr/local/share/postgresql/extension/postgis* /to-host/share/ 2>/dev/null || true"

# Копируем файлы из временной директории хоста в образ YugabyteDB
# (Этот шаг требует, чтобы на хосте был Docker и он мог запускать контейнеры во время сборки)
COPY --from=postgis/postgis:11-3.1 \
     /usr/local/lib/postgresql/postgis* /home/yugabyte/postgres/lib/
COPY --from=postgis/postgis:11-3.1 \
     /usr/local/share/postgresql/extension/postgis* /home/yugabyte/postgres/share/extension/

RUN /home/yugabyte/bin/post_install.sh -e
EOF

# Соберите альтернативный образ
docker build -f Dockerfile.alternative -t yugabyte-with-postgis-alt:latest .
```

> **⚠️ Примечание:** Второй подход (`Dockerfile.alternative`) требует, чтобы Docker на вашем хосте поддерживал **функцию сборки из другого образа (`COPY --from`)** для нестатических файлов. Иногда это может не сработать из-за разницы в путях. Первый способ с исправлением репозитория — более прямой и предпочтительный.

### 💎 Рекомендация
1.  **Попробуйте исправленный Dockerfile** (первый вариант). Шансы на успех высоки.
2.  **Если возникает ошибка о конфликте библиотек**, попробуйте добавить в этап builder очистку старых версий (как советуется в статьях о CentOS 7), но это может усложнить процесс.
3.  В крайнем случае **можно установить PostGIS вручную в запущенный контейнер** для теста, следуя документации YugabyteDB, но это временное решение.
