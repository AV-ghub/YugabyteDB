```
docker build -t yugabyte-with-postgis-test:latest .
Sending build context to Docker daemon  9.216kB
Step 1/9 : FROM yugabytedb/yugabyte:2.15.3.2-b1 as postgis-builder
 ---> 85aae308f23a
Step 2/9 : RUN sed -i 's|^mirrorlist=|#mirrorlist=|g' /etc/yum.repos.d/CentOS-*.repo &&     sed -i 's|^#baseurl=http://mirror.centos.org/centos|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*.repo
 ---> Using cache
 ---> deb0a05f629d
Step 3/9 : RUN yum-config-manager --disable epel 2>/dev/null || true &&     yum-config-manager --disable epel-source 2>/dev/null || true
 ---> Running in 78bad8df944a
Loaded plugins: fastestmirror, ovl
================================== repo: epel ==================================
[epel]
async = True
bandwidth = 0
base_persistdir = /var/lib/yum/repos/x86_64/7
baseurl =
cache = 0
cachedir = /var/cache/yum/x86_64/7/epel
check_config_file_age = True
compare_providers_priority = 80
cost = 1000
deltarpm_metadata_percentage = 100
deltarpm_percentage =
enabled = 0
enablegroups = True
exclude =
failovermethod = priority
ftp_disable_epsv = False
gpgcadir = /var/lib/yum/repos/x86_64/7/epel/gpgcadir
gpgcakey =
gpgcheck = True
gpgdir = /var/lib/yum/repos/x86_64/7/epel/gpgdir
gpgkey = file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
hdrdir = /var/cache/yum/x86_64/7/epel/headers
http_caching = all
includepkgs =
ip_resolve =
keepalive = True
keepcache = False
mddownloadpolicy = sqlite
mdpolicy = group:small
mediaid =
metadata_expire = 21600
metadata_expire_filter = read-only:present
metalink = https://mirrors.fedoraproject.org/metalink?repo=epel-7&arch=x86_64&infra=container&content=centos
minrate = 0
mirrorlist =
mirrorlist_expire = 86400
name = Extra Packages for Enterprise Linux 7 - x86_64
old_base_cache_dir =
password =
persistdir = /var/lib/yum/repos/x86_64/7/epel
pkgdir = /var/cache/yum/x86_64/7/epel/packages
proxy = False
proxy_dict =
proxy_password =
proxy_username =
repo_gpgcheck = False
retries = 10
skip_if_unavailable = False
ssl_check_cert_permissions = True
sslcacert =
sslclientcert =
sslclientkey =
sslverify = True
throttle = 0
timeout = 30.0
ui_id = epel/x86_64
ui_repoid_vars = releasever,
   basearch
username =

Loaded plugins: fastestmirror, ovl
============================== repo: epel-source ===============================
[epel-source]
async = True
bandwidth = 0
base_persistdir = /var/lib/yum/repos/x86_64/7
baseurl =
cache = 0
cachedir = /var/cache/yum/x86_64/7/epel-source
check_config_file_age = True
compare_providers_priority = 80
cost = 1000
deltarpm_metadata_percentage = 100
deltarpm_percentage =
enabled = False
enablegroups = True
exclude =
failovermethod = priority
ftp_disable_epsv = False
gpgcadir = /var/lib/yum/repos/x86_64/7/epel-source/gpgcadir
gpgcakey =
gpgcheck = True
gpgdir = /var/lib/yum/repos/x86_64/7/epel-source/gpgdir
gpgkey = file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
hdrdir = /var/cache/yum/x86_64/7/epel-source/headers
http_caching = all
includepkgs =
ip_resolve =
keepalive = True
keepcache = False
mddownloadpolicy = sqlite
mdpolicy = group:small
mediaid =
metadata_expire = 21600
metadata_expire_filter = read-only:present
metalink = https://mirrors.fedoraproject.org/metalink?repo=epel-source-7&arch=x86_64&infra=container&content=centos
minrate = 0
mirrorlist =
mirrorlist_expire = 86400
name = Extra Packages for Enterprise Linux 7 - x86_64 - Source
old_base_cache_dir =
password =
persistdir = /var/lib/yum/repos/x86_64/7/epel-source
pkgdir = /var/cache/yum/x86_64/7/epel-source/packages
proxy = False
proxy_dict =
proxy_password =
proxy_username =
repo_gpgcheck = False
retries = 10
skip_if_unavailable = False
ssl_check_cert_permissions = True
sslcacert =
sslclientcert =
sslclientkey =
sslverify = True
throttle = 0
timeout = 30.0
ui_id = epel-source/x86_64
ui_repoid_vars = releasever,
   basearch
username =

Removing intermediate container 78bad8df944a
 ---> d9d6869ad2c4
Step 4/9 : RUN yum install -y wget &&     wget https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm &&     rpm -ivh pgdg-redhat-repo-latest.noarch.rpm &&     yum install -y postgresql11-server postgis31_11 postgis31_11-client
 ---> Running in f112eb267dd7
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
http://vault.centos.org/7/os/x86_64/repodata/repomd.xml: [Errno 14] HTTPS Error 301 - Moved Permanently
Trying other mirror.


 One of the configured repositories failed (CentOS-7 - Base),
 and yum doesn't have enough cached data to continue. At this point the only
 safe thing yum can do is fail. There are a few ways to work "fix" this:

     1. Contact the upstream for the repository and get them to fix the problem.

     2. Reconfigure the baseurl/etc. for the repository, to point to a working
        upstream. This is most often useful if you are using a newer
        distribution release than is supported by the repository (and the
        packages for the previous distribution release still work).

     3. Run the command with the repository temporarily disabled
            yum --disablerepo=base ...

     4. Disable the repository permanently, so yum won't use it by default. Yum
        will then just ignore the repository until you permanently enable it
        again or use --enablerepo for temporary usage:

            yum-config-manager --disable base
        or
            subscription-manager repos --disable=base

     5. Configure the failing repository to be skipped, if it is unavailable.
        Note that yum will try to contact the repo. when it runs most commands,
        so will have to try and fail each time (and thus. yum will be be much
        slower). If it is a very temporary problem though, this is often a nice
        compromise:

            yum-config-manager --save --setopt=base.skip_if_unavailable=true

failure: repodata/repomd.xml from base: [Errno 256] No more mirrors to try.
http://vault.centos.org/7/os/x86_64/repodata/repomd.xml: [Errno 14] HTTPS Error 301 - Moved Permanently
The command '/bin/sh -c yum install -y wget &&     wget https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm &&     rpm -ivh pgdg-redhat-repo-latest.noarch.rpm &&     yum install -y postgresql11-server postgis31_11 postgis31_11-client' returned a non-zero code: 1

```

Ошибка указывает, что проблема перешла на репозиторий `base`. Судя по всему, архив `vault.centos.org`, на который мы перенаправили запросы, не отвечает или перенаправляет некорректно. Это известная трудность с устаревшим образом CentOS 7 . Более эффективным решением будет **временно отключить все репозитории CentOS** во время сборки PostGIS, так как они нам для этой цели не нужны.

Вот исправленная инструкция:

1.  **Замените файл `Dockerfile`** в вашей рабочей директории (`~/yugabyte-postgis-test`) на новую, более эффективную версию:

    ```bash
    cat > Dockerfile << 'EOF'
    # Первый этап: Установка PostGIS в среде, совместимой с YugabyteDB
    FROM yugabytedb/yugabyte:2.15.3.2-b1 as postgis-builder

    # Ключевое исправление: отключаем ВСЕ репозитории по умолчанию для 'yum install'.
    # Это позволит установить только те пакеты, которые есть в добавляемом нами репозитории PostgreSQL.
    RUN yum install -y wget && \
        wget https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm && \
        rpm -ivh pgdg-redhat-repo-latest.noarch.rpm && \
        # '--disablerepo=*' отключает все репозитории, '--enablerepo=pgdg-common' включает только нужный
        yum --disablerepo=* --enablerepo=pgdg-common install -y postgresql11-server postgis31_11 postgis31_11-client

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

2.  **Попробуйте собрать образ снова**:

    ```bash
    docker build -t yugabyte-with-postgis-test:latest .
    ```

**Объяснение ключевого изменения:**
*   Флаги `--disablerepo=*` и `--enablerepo=pgdg-common` в команде `yum install` гарантируют, что система не будет пытаться искать пакеты в сломанных репозиториях CentOS 7. Она установит PostGIS 3.1 для PostgreSQL 11 и все его зависимости напрямую из официального репозитория PostgreSQL (PGDG), что должно решить проблему с сетью .

> ⚠️ **Что делать, если возникнут новые ошибки зависимостей?**
> Если при установке `postgis31_11` появятся ошибки о конфликтах версий библиотек (например, `geos` или `gdal`), это распространённая проблема . Это означает, что в системе уже есть более старые версии библиотек от CentOS. В этом случае может помочь полное удаление существующих версий геобиблиотек из образа перед установкой. Вы можете добавить в `Dockerfile` перед установкой PostGIS следующие команды:
> ```bash
> # Внутри этапа 'postgis-builder', после установки wget, но перед установкой PostGIS
> yum remove -y geos* gdal* proj* 2>/dev/null || true
> ```
> После этого повторите попытку установки PostGIS с использованием флагов `--disablerepo`.
