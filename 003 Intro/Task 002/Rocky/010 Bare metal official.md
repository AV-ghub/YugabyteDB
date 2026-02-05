```
скачал вручную отсюда https://download.yugabyte.com/
распаковал и пытаюсь пустить
./bin/yugabyted start
Starting yugabyted...
/ Starting the YugabyteDB Processes...Failed running /_a/yugabyte-2025.2.0.1/bin/post_install.sh (exit code: 126). Standard output:
b'OpenSSL binary: /_a/yugabyte-2025.2.0.1/bin/../bin/../bin/openssl\nFIPS module: /_a/yugabyte-2025.2.0.1/bin/../bin/../lib/ossl-modules/fips.dylib\n'
. Standard error:
b'/_a/yugabyte-2025.2.0.1/bin/../bin/fips_install.sh: line 41: /_a/yugabyte-2025.2.0.1/bin/../bin/../bin/openssl: cannot execute binary file: Exec format error\n'
For more information, check the logs in /root/var/logs

в логах
tail -n 50 /root/var/logs/yugabyted.log
[yugabyted start] 2026-02-05 13:19:53,630 INFO:  | 0.0s | Found directory /_a/yugabyte-2025.2.0.1/bin for file openssl_proxy.sh
[yugabyted start] 2026-02-05 13:19:53,630 INFO:  | 0.0s | Found directory /_a/yugabyte-2025.2.0.1/bin for file yb-admin
[yugabyted start] 2026-02-05 13:19:53,630 INFO:  | 0.0s | Found directory /_a/yugabyte-2025.2.0.1/bin for file yb-ts-cli
[yugabyted start] 2026-02-05 13:19:53,630 INFO:  | 0.0s | Found directory /_a/yugabyte-2025.2.0.1/postgres/bin for file pg_upgrade
[yugabyted start] 2026-02-05 13:19:53,639 INFO:  | 0.0s | Starting first primary node. Using b46e5449-40c6-4d3f-80d8-b1dc487858b8 as placement_uuid
[yugabyted start] 2026-02-05 13:19:53,641 INFO:  | 0.0s | Starting yugabyted...
[yugabyted start] 2026-02-05 13:19:53,645 INFO:  | 0.0s | Daemon grandchild process begins execution.
[yugabyted start] 2026-02-05 13:19:53,645 INFO:  | 0.0s | yugabyted started running with PID 208291.
[yugabyted start] 2026-02-05 13:19:53,647 ERROR:  | 0.0s | Error changing RLIMIT_NOFILE from 1024 to 1048576: current limit exceeds maximum limit
[yugabyted start] 2026-02-05 13:19:53,648 INFO:  | 0.0s | Found directory /_a/yugabyte-2025.2.0.1/bin for file configure_clockbound.sh
[yugabyted start] 2026-02-05 13:19:53,659 INFO:  | 0.1s | run_process: cmd: ['chronyc', 'sources']
[yugabyted start] 2026-02-05 13:19:54,664 INFO:  | 1.1s | run_process: ['chronyc', 'sources'] timeout expired for command:
[yugabyted start] 2026-02-05 13:19:54,666 INFO:  | 1.1s | Found directory /_a/yugabyte-2025.2.0.1/bin for file yb-master
[yugabyted start] 2026-02-05 13:19:54,666 DEBUG:  | 1.1s | Value of saved_data - upgrade_in_progress. False
[yugabyted start] 2026-02-05 13:19:54,666 DEBUG:  | 1.1s | Value of saved_data - catalog_upgrade_completed. False
[yugabyted start] 2026-02-05 13:19:54,667 INFO:  | 1.1s | Found directory /_a/yugabyte-2025.2.0.1/bin for file yb-tserver
[yugabyted start] 2026-02-05 13:19:54,669 INFO:  | 1.1s | Found directory /_a/yugabyte-2025.2.0.1/bin for file post_install.sh
[yugabyted start] 2026-02-05 13:19:54,669 INFO:  | 1.1s | Running the post-installation script /_a/yugabyte-2025.2.0.1/bin/post_install.sh (may be a no-op)
[yugabyted start] 2026-02-05 13:19:54,683 ERROR:  | 1.1s | Failed running /_a/yugabyte-2025.2.0.1/bin/post_install.sh (exit code: 126). Standard output:
b'OpenSSL binary: /_a/yugabyte-2025.2.0.1/bin/../bin/../bin/openssl\nFIPS module: /_a/yugabyte-2025.2.0.1/bin/../bin/../lib/ossl-modules/fips.dylib\n'
. Standard error:
b'/_a/yugabyte-2025.2.0.1/bin/../bin/fips_install.sh: line 41: /_a/yugabyte-2025.2.0.1/bin/../bin/../bin/openssl: cannot execute binary file: Exec format error\n'

на что это может указывать?
```

Это указывает на **критическую проблему: вы скачали не ту версию архива**.

Ошибка `fips.dylib` и `cannot execute binary file: Exec format error` означает, что вы скачали **сборку для macOS (Darwin/Apple Silicon)**, а не для Linux. Файл `fips.dylib` — это динамическая библиотека macOS, а на Linux должны быть `.so` файлы.

### 🔍 Проверка и исправление

1.  **Проверьте, что вы скачали**:
    ```bash
    # Посмотрите на имя файла
    ls -la yugabyte-*.tar.gz
    # Правильное имя для Linux: `yugabyte-2025.2.0.1-linux-x86_64.tar.gz`
    # Неправильное (для macOS): `yugabyte-2025.2.0.1-darwin-arm64.tar.gz` или `...-darwin-x86_64.tar.gz`
    ```

2.  **Скачайте правильную версию для Linux x86_64**:
    ```bash
    # Удалите macOS-версию
    rm -rf yugabyte-2025.2.0.1* yugabyte-*.tar.gz

    # СКАЧАЙТЕ ИМЕННО ЭТУ ВЕРСИЮ (Linux x86_64)
    # Попробуйте прямую ссылку:
    wget https://downloads.yugabyte.com/releases/2025.2.0.1/yugabyte-2025.2.0.1-linux-x86_64.tar.gz

    # Если не сработает, попробуйте через curl с игнорированием SSL:
    curl -k -L -o yugabyte-linux.tar.gz https://downloads.yugabyte.com/releases/2025.2.0.1/yugabyte-2025.2.0.1-linux-x86_64.tar.gz
    ```

3.  **Распакуйте и запустите**:
    ```bash
    tar -xzf yugabyte-2025.2.0.1-linux-x86_64.tar.gz
    cd yugabyte-2025.2.0.1
    ./bin/post_install.sh
    ./bin/yugabyted start --background=false
    ```

### ⚡ Альтернатива: Быстрая проверка через уже имеющийся Docker
Пока скачивается правильная версия, вы можете **запустить YugabyteDB прямо из Docker-контейнера в режиме "хоста"**, что даст вам доступ к процессу на локальной машине:

```bash
# Запустите контейнер, монтируя его бинарники как volume
docker run -d \
  --name yb-host-mode \
  --network host \
  -v /tmp/yugabyte-data:/home/yugabyte/yb_data \
  yugabytedb/yugabyte:2025.2.0.1-b1 \
  /home/yugabyte/bin/yugabyted start --background=false --listen=0.0.0.0

# Подключитесь (порт 5433 будет доступен на localhost)
sleep 90
docker exec yb-host-mode /home/yugabyte/bin/ysqlsh -h localhost -p 5433 -U yugabyte -c "SELECT version();"
```

### 📋 Что делать дальше
1.  **Скачайте правильный Linux-архив** по ссылке выше.
2.  **Если скачать не получается** из-за сетевых проблем — используйте Docker-метод как временное решение.
3.  **После успешного запуска YugabyteDB** продолжайте установку PostGIS по плану:
    ```bash
    # Установите PostGIS из репозиториев Rocky Linux
    sudo dnf install -y postgresql15-server postgis35_15

    # Скопируйте файлы PostGIS в каталог YugabyteDB
    # (остановив yugabyted перед копированием)
    sudo cp -r /usr/pgsql-15/lib/postgis* /path/to/yugabyte-2025.2.0.1/postgres/lib/
    sudo cp -r /usr/share/pgsql-15/extension/postgis* /path/to/yugabyte-2025.2.0.1/postgres/share/extension/
    ```

**Скачали ли вы архив с названием, содержащим `linux-x86_64`?** Если да, и проблема осталась — возможно, архив повреждён. Если не можете скачать — Docker-метод даст вам рабочую среду за 2 минуты.
