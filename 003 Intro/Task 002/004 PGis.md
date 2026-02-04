```
docker build -t yugabyte-with-postgis-test:latest .
Sending build context to Docker daemon  6.144kB
Step 1/8 : FROM yugabytedb/yugabyte:2.15.3.2-b1 as postgis-builder
 ---> 85aae308f23a
Step 2/8 : RUN sed -i 's|^mirrorlist=|#mirrorlist=|g' /etc/yum.repos.d/CentOS-*.repo &&     sed -i 's|^#baseurl=http://mirror.centos.org/centos|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*.repo
 ---> Running in bee4f480c26a
Removing intermediate container bee4f480c26a
 ---> deb0a05f629d
Step 3/8 : RUN yum update -y &&     yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm &&     yum install -y postgresql11-server postgis31_11 postgis31_11-client
 ---> Running in 3f530a2fd8d6
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors


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

Cannot retrieve metalink for repository: epel/x86_64. Please verify its path and try again
The command '/bin/sh -c yum update -y &&     yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm &&     yum install -y postgresql11-server postgis31_11 postgis31_11-client' returned a non-zero code: 1

```

Хорошо, проблема перешла на репозиторий **EPEL** (Extra Packages for Enterprise Linux). Нужно его тоже исправить. CentOS 7 официально завершил поддержку, и многие его репозитории переехали или закрылись.

### 🔧 Окончательно исправленный Dockerfile

Основная проблема в том, что `yum update -y` пытается обновить пакеты из всех репозиториев, включая `epel`. Давайте отключим обновление системы (оно нам не критично) и сфокусируемся на установке только PostGIS. Также исправим все основные репозитории.

Замените содержимое файла `Dockerfile` на этот вариант:

```bash
cat > Dockerfile << 'EOF'
# Первый этап: Установка PostGIS в среде, совместимой с YugabyteDB
FROM yugabytedb/yugabyte:2.15.3.2-b1 as postgis-builder

# 1. Полностью исправляем репозитории CentOS 7 (основные + epel)
RUN sed -i 's|^mirrorlist=|#mirrorlist=|g' /etc/yum.repos.d/CentOS-*.repo && \
    sed -i 's|^#baseurl=http://mirror.centos.org/centos|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*.repo

# 2. Отключаем репозиторий epel, если он есть (чтобы не мешал)
RUN yum-config-manager --disable epel 2>/dev/null || true && \
    yum-config-manager --disable epel-source 2>/dev/null || true

# 3. Устанавливаем только PostGIS и его зависимости. Пропускаем 'yum update'.
RUN yum install -y wget && \
    wget https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm && \
    rpm -ivh pgdg-redhat-repo-latest.noarch.rpm && \
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

**Ключевые изменения:**
1.  **Убран `yum update -y`** — для сборки PostGIS он не нужен.
2.  **Явно отключены репозитории EPEL**, которые вызывают ошибку.
3.  **Заменён способ установки** — сначала ставится `wget`, затем скачивается и устанавливается репозиторий PostgreSQL, и только потом PostGIS.

### 🚀 Сборка и тест
Попробуйте собрать образ снова:

```bash
# 1. Соберите исправленный образ
docker build -t yugabyte-with-postgis-test:latest .

# 2. Если сборка прошла успешно, запустите тестовый контейнер
docker run -d \
  --name yb-single-node-test \
  -p 5433:5433 \
  -p 9042:9042 \
  -p 7000:7000 \
  yugabyte-with-postgis-test:latest \
  bin/yugabyted start \
  --base_dir=/home/yugabyte/yb_data \
  --daemon=false

# Дайте время на запуск
sleep 30

# 3. Проверьте работу PostGIS
docker exec -it yb-single-node-test ./bin/ysqlsh -h localhost -p 5433 -U yugabyte << 'SQL'
CREATE EXTENSION postgis;
SELECT PostGIS_Full_Version();
SQL
```

### ⚠️ Если ошибка появится снова
Если вы увидите ошибку, связанную с **отсутствующими зависимостями** (например, `libgdal.so` или `libgeos.so`), это значит, что версии библиотек в репозитории CentOS 7 слишком старые. В этом случае нам нужно использовать более новый базовый образ для установки PostGIS.

Замените **первую строку** `Dockerfile` на образ с более свежей системой, но той же архитектурой, например, на Rocky Linux 8:

```dockerfile
# Первый этап: Установка PostGIS в современной среде
FROM rockylinux:8 as postgis-builder
```

Однако это создаст **новую проблему**: библиотеки из Rocky Linux 8 могут быть несовместимы со старыми библиотеками в финальном образе YugabyteDB (который основан на AlmaLinux 8). Возможно, придётся копировать не только файлы PostGIS, но и недостающие библиотеки.

### 💡 Альтернатива: Ручная установка в тестовом контейнере (быстрый чек)
Если сборка образа всё ещё вызывает сложности, а вам нужно быстро проверить концепцию, можно установить PostGIS **вручную прямо в запущенный контейнер** вашего текущего кластера YugabyteDB.

Вернитесь к контейнеру `yugabyte_yb-tserver1_1` и выполните команды, которые мы обсуждали ранее:

```bash
# Подключитесь к контейнеру T-Server
docker exec -it yugabyte_yb-tserver1_1 bash

# Внутри контейнера попробуйте ручное исправление (из наших старых обсуждений):
# 1. Скачайте и установите совместимые библиотеки
# 2. Скопируйте файлы PostGIS из временного контейнера
# 3. Выполните /home/yugabyte/bin/post_install.sh -e
```

Это даст понимание, работает ли PostGIS в принципе с вашей версией YugabyteDB, без долгой сборки образа.
