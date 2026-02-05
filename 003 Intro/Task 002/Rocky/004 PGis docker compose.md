```
в итоге докер запустился по первому сценарию со сбросом конфигурации
причем daemon.json поставил изначальный такого плана
{
    "dns": [
        "хх.хх.хх.хх",
        "хх.хх.хх.хх"
    ],
    "insecure-registries": [
        "ххххххххххх.com:5000",
        "ххххххххххх.com:5000",
        "ххххххххххх.com"
    ],
    "labels": [
        "role=infra"
    ],
    "ipv6": false,
    "data-root": "/data/docker",
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "30m",
        "max-file": "50"
    }
}
с ним тоже все запустилось примерно так
[]# sudo systemctl status docker --no-pager
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: disabled)
     Active: active (running) since Thu 2026-02-05 06:47:39 UTC; 8s ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 178728 (dockerd)
      Tasks: 16
     Memory: 33.0M
        CPU: 444ms
     CGroup: /system.slice/docker.service
             └─178728 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Feb 05 06:47:38  dockerd[178728]: time="2026-02-05T06:47:38.344253581Z" level=info msg="Restoring containers: start."
Feb 05 06:47:38  dockerd[178728]: time="2026-02-05T06:47:38.359735171Z" level=info msg="Deleting nftables IPv4 rules" error="exit status 1"
Feb 05 06:47:38  dockerd[178728]: time="2026-02-05T06:47:38.375496080Z" level=info msg="Deleting nftables IPv6 rules" error="exit status 1"
Feb 05 06:47:38  dockerd[178728]: time="2026-02-05T06:47:38.989274652Z" level=info msg="Loading containers: done."
Feb 05 06:47:39  dockerd[178728]: time="2026-02-05T06:47:39.001897524Z" level=info msg="Docker daemon" commit=e9ff10b containerd-snapshotter=false storage-driver=overla… version=29.0.2
Feb 05 06:47:39  dockerd[178728]: time="2026-02-05T06:47:39.002004335Z" level=info msg="Initializing buildkit"
Feb 05 06:47:39  dockerd[178728]: time="2026-02-05T06:47:39.027433710Z" level=info msg="Completed buildkit initialization"
Feb 05 06:47:39  dockerd[178728]: time="2026-02-05T06:47:39.038107903Z" level=info msg="Daemon has completed initialization"
Feb 05 06:47:39  dockerd[178728]: time="2026-02-05T06:47:39.038251686Z" level=info msg="API listen on /run/docker.sock"
Feb 05 06:47:39  systemd[1]: Started Docker Application Container Engine.
Hint: Some lines were ellipsized, use -l to show in full.
[]#
[]#
[]# docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

вопрос только что там пара ошибок, но они были и с минимальным daemon.json
насколько это критично?
и что тогда делаем дальше?
с композом пока такая же ситуация
docker compose ps
no configuration file provided: not found
```

Отлично! Docker запущен. Ошибки `"Deleting nftables IPv6 rules" error="exit status 1"` **не критичны** — это просто сообщения о том, что Docker пытался удалить старые правила сетевого фаервола (nftables) для IPv6, но их не нашёл. На работу Docker это не влияет.

Теперь переходим к следующему шагу — **Docker Compose**.

## 🔧 Исправление проблемы с Docker Compose

Ошибка `no configuration file provided: not found` означает, что Docker Compose ищет файл конфигурации в текущей директории, но не находит его. **Это нормально**, когда мы просто проверяем команду без контекста проекта.

### 1. **Создайте тестовый docker-compose.yml**

```bash
# Создайте директорию для теста
mkdir yugabyte-test
cd yugabyte-test

# Создайте простейший docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  yb-master:
    image: yugabytedb/yugabyte:latest
    container_name: yb-master
    command: ["/home/yugabyte/bin/yb-master",
              "--fs_data_dirs=/mnt/data",
              "--master_addresses=yb-master:7100",
              "--rpc_bind_addresses=yb-master:7100",
              "--enable_ysql=true",
              "--callhome_enabled=false"]
    ports:
      - "7000:7000"
    volumes:
      - yb-master-data:/mnt/data

  yb-tserver:
    image: yugabytedb/yugabyte:latest
    container_name: yb-tserver
    command: ["/home/yugabyte/bin/yb-tserver",
              "--fs_data_dirs=/mnt/data",
              "--tserver_master_addrs=yb-master:7100",
              "--rpc_bind_addresses=yb-tserver:9100",
              "--enable_ysql=true",
              "--callhome_enabled=false"]
    ports:
      - "5433:5433"
      - "9042:9042"
      - "9000:9000"
    volumes:
      - yb-tserver-data:/mnt/data
    depends_on:
      - yb-master

volumes:
  yb-master-data:
  yb-tserver-data:
EOF
```

### 2. **Проверьте работу Docker Compose**

```bash
# 1. Проверьте версию Compose (она уже установлена)
docker compose version

# 2. Запустите кластер в фоновом режиме
docker compose up -d

# 3. Проверьте статус контейнеров
docker compose ps
docker ps

# 4. Дайте время на запуск (30 секунд)
sleep 30

# 5. Проверьте подключение к YSQL
docker exec yb-tserver /home/yugabyte/bin/ysqlsh -h localhost -p 5433 -U yugabyte -c "SELECT version();"
```

### 3. **Если возникнут проблемы с сетью в Compose**

Ваш `daemon.json` содержит корпоративные DNS и `insecure-registries`. Это может влиять на скачивание образов. Проверьте:

```bash
# Попробуйте скачать образ вручную
docker pull yugabytedb/yugabyte:latest

# Если будет ошибка SSL/сети, попробуйте временно добавить:
echo '{"insecure-registries": ["docker.io", "registry-1.docker.io"]}' | sudo tee -a /etc/docker/daemon.json
sudo systemctl restart docker
```

## 📋 План действий на сегодня

1. **✅ Docker запущен и работает**
2. **🔧 Настроить Docker Compose** (текущий этап)
3. **🚀 Развернуть кластер YugabyteDB**
4. **🎯 Вернуться к PostGIS** (на новой виртуалке)

### Если Docker Compose не находит образы:

```bash
# Попробуйте указать репозиторий явно
image: docker.io/yugabytedb/yugabyte:latest

# Или используйте docker pull заранее
docker pull yugabytedb/yugabyte:latest
```

## 🎯 Что дальше после успешного запуска Compose

Если кластер запустится успешно, мы сможем:

1. **Протестировать оба интерфейса** (YSQL и YCQL) как делали ранее
2. **Вернуться к PostGIS** — теперь на современной системе без проблем CentOS 7
3. **Собрать образ с PostGIS** используя актуальные репозитории Rocky Linux 9

**Ключевой момент**: На Rocky Linux 9 сборка PostGIS будет значительно проще, так как репозитории PostgreSQL и PostGIS там актуальны и работают.

Попробуйте запустить тестовый docker-compose.yml. Если всё сработает — приступаем к развёртыванию полноценного кластера. Если будут ошибки — пришлите вывод команд `docker compose up -d` и `docker compose logs`.




----------------------------------------------------------------




```
все скачалось и создалось, но работать пока отказывается
docker exec yb-tserver /home/yugabyte/bin/ysqlsh -h localhost -p 5433 -U yugabyte -c "SELECT version();"
ysqlsh: error: connection to server at "localhost" (::1), port 5433 failed: Connection refused
        Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 5433 failed: Connection refused
        Is the server running on that host and accepting TCP/IP connections?
```

Проблема в том, что контейнеры запустились, но **процесс YugabyteDB внутри них ещё не полностью инициализировался**. Нужно дать им больше времени и проверить логи.

## 🔍 Диагностика и решение

### 1. **Проверьте статус контейнеров и подождите**

```bash
# 1. Посмотрите логи T-Server (самый информативный)
docker compose logs yb-tserver --tail 50

# 2. Посмотрите логи Master
docker compose logs yb-master --tail 30

# 3. Проверьте, запущены ли процессы внутри контейнеров
docker exec yb-tserver ps aux | grep -E '(yb-tserver|postgres)'
docker exec yb-master ps aux | grep yb-master

# 4. Дайте больше времени на инициализацию (может занять 1-2 минуты)
sleep 90

# 5. Повторите попытку подключения
docker exec yb-tserver /home/yugabyte/bin/ysqlsh -h localhost -p 5433 -U yugabyte -c "SELECT version();"
```

### 2. **Если всё ещё не работает — упростим конфигурацию**

Сложная конфигурация с раздельными master и tserver может требовать тонкой настройки. Давайте **запустим одноузловой кластер** через `yugabyted` — это проще и надёжнее для тестов.

Создайте новый файл `docker-compose-single.yml`:

```bash
cat > docker-compose-single.yml << 'EOF'
version: '3.8'
services:
  yugabyte:
    image: yugabytedb/yugabyte:latest
    container_name: yb-single-node
    command: ["/home/yugabyte/bin/yugabyted", "start", "--background=false", "--daemon=false"]
    ports:
      - "5433:5433"
      - "9042:9042"
      - "7000:7000"
      - "9000:9000"
    volumes:
      - yb-data:/home/yugabyte/yb_data
    environment:
      - YB_MASTER_RPC_PORT=7100
      - YB_TSERVER_RPC_PORT=9100
    healthcheck:
      test: ["CMD", "/home/yugabyte/bin/yugabyted", "status"]
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  yb-data:
EOF
```

Запустите его:

```bash
# Остановите старый кластер (если запущен)
docker compose down -v

# Запустите новый одноузловой
docker compose -f docker-compose-single.yml up -d

# Подождите 60-90 секунд
sleep 90

# Проверьте статус через yugabyted
docker exec yb-single-node /home/yugabyte/bin/yugabyted status

# Попробуйте подключиться
docker exec yb-single-node /home/yugabyte/bin/ysqlsh -h localhost -p 5433 -U yugabyte -c "SELECT version();"
```

### 3. **Прямая проверка портов внутри контейнера**

Если подключение всё ещё не работает, проверьте, слушает ли процесс порты:

```bash
# 1. Зайдите внутрь контейнера
docker exec -it yb-single-node bash

# 2. Внутри контейнера проверьте:
# Проверьте, запущен ли yugabyted
ps aux | grep yugabyted

# Проверьте, слушает ли порт 5433
netstat -tlnp | grep 5433

# Проверьте, есть ли процесс postgres
ps aux | grep postgres

# Попробуйте подключиться локально изнутри контейнера
./bin/ysqlsh -h 127.0.0.1 -p 5433 -U yugabyte

# Если не работает, проверьте логи
tail -100 /home/yugabyte/yb_data/node-1/disk-1/yb-data/tserver/logs/postgresql-*.log
```

### 4. **Альтернативный вариант: быстрый тест через docker run**

Если compose всё ещё вызывает проблемы, протестируйте напрямую:

```bash
# Запустите одноузловой кластер одной командой
docker run -d \
  --name yb-test \
  -p 5433:5433 \
  -p 9042:9042 \
  -p 7000:7000 \
  yugabytedb/yugabyte:latest \
  /home/yugabyte/bin/yugabyted start \
  --base_dir=/home/yugabyte/yb_data \
  --daemon=false

# Подождите
sleep 120

# Проверьте
docker exec yb-test /home/yugabyte/bin/yugabyted status
docker exec yb-test /home/yugabyte/bin/ysqlsh -c "SELECT version();"
```

## 📊 Частые причины и решения

| Симптом | Причина | Решение |
|---------|---------|---------|
| **Connection refused** | Процесс не запущен или не готов | Дать больше времени (60-120 сек) |
| **Порт 5433 не слушается** | YSQL не включен или ошибка конфигурации | Проверить флаг `--enable_ysql=true` |
| **Master и T-Server не видят друг друга** | Проблемы с сетью Docker Compose | Использовать одноузловую конфигурацию |
| **Ошибка в логах про диски** | Проблемы с volumes | Проверить права на директории |

## 🎯 Рекомендация

**Начните с шага 2** (одноузловой кластер через `yugabyted`). Это самый надёжный способ для быстрого тестирования. Если он заработает — у вас будет рабочая основа для тестирования PostGIS.

Попробуйте `docker-compose-single.yml`. Если будут ошибки — пришлите вывод команд `docker compose logs` и `docker exec yb-single-node /home/yugabyte/bin/yugabyted status`.




---------------------------------------------------------------






```
вот похоже в чем дело
docker run -d \
  --name yb-test \
  -p 5433:5433 \
  -p 9042:9042 \
  -p 7000:7000 \
  yugabytedb/yugabyte:latest \
  /home/yugabyte/bin/yugabyted start \
  --base_dir=/home/yugabyte/yb_data \
  --daemon=false
b29d636b18e8b7532bd6a598074f20c5341522f33395d9e0d00bbb94d9114220
docker: Error response from daemon: failed to set up container networking: driver failed programming external connectivity on endpoint yb-test (2543da41562181927a92b80467032db8f10deb168ec22c13401c3c6d7b0551d5): Bind for 0.0.0.0:5433 failed: port is already allocated
```

Проблема ясна — **порт 5433 уже занят** на вашей хост-машине. Это скорее всего осталось от предыдущих запусков Docker Compose или других PostgreSQL-сервисов.

## 🔧 Решение: Освобождение портов или использование других портов

### 1. **Найдите, что занимает порты**

```bash
# Проверьте, какие процессы используют порты 5433, 9042, 7000
sudo netstat -tlnp | grep -E ':(5433|9042|7000)'

# Альтернативно, используйте ss (более современная утилита)
sudo ss -tlnp | grep -E ':(5433|9042|7000)'

# Если хотите увидеть все Docker-контейнеры и их порты
docker ps -a --format "table {{.Names}}\t{{.Ports}}"
```

### 2. **Остановите конкурирующие контейнеры**

```bash
# Остановите ВСЕ контейнеры (если они не нужны)
docker stop $(docker ps -q) 2>/dev/null || true

# Удалите остановленные контейнеры
docker rm $(docker ps -a -q) 2>/dev/null || true

# Остановите Docker Compose проекты
docker compose down 2>/dev/null || true
docker compose -f docker-compose-single.yml down 2>/dev/null || true
```

### 3. **Используйте альтернативные порты** (самый быстрый способ)

Просто измените порты при запуске:

```bash
# Запустите с другими портами (например, 5434, 9043, 7001)
docker run -d \
  --name yb-test \
  -p 5434:5433 \   # YSQL на порту 5434 вместо 5433
  -p 9043:9042 \   # YCQL на порту 9043 вместо 9042
  -p 7001:7000 \   # Web UI на порту 7001 вместо 7000
  yugabytedb/yugabyte:latest \
  /home/yugabyte/bin/yugabyted start \
  --base_dir=/home/yugabyte/yb_data \
  --daemon=false

# Подождите 60 секунд
sleep 60

# Подключитесь через новый порт
docker exec yb-test /home/yugabyte/bin/ysqlsh -h localhost -p 5433 -U yugabyte -c "SELECT version();"
```

### 4. **Обновленный Docker Compose с альтернативными портами**

Создайте `docker-compose-alt-ports.yml`:

```bash
cat > docker-compose-alt-ports.yml << 'EOF'
version: '3.8'
services:
  yugabyte:
    image: yugabytedb/yugabyte:latest
    container_name: yb-single-alt
    command: ["/home/yugabyte/bin/yugabyted", "start", "--background=false", "--daemon=false"]
    ports:
      - "5434:5433"   # YSQL
      - "9043:9042"   # YCQL
      - "7001:7000"   # Web UI
      - "9001:9000"   # Admin UI
    volumes:
      - yb-data-alt:/home/yugabyte/yb_data
    healthcheck:
      test: ["CMD", "/home/yugabyte/bin/yugabyted", "status"]
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  yb-data-alt:
EOF
```

Запустите его:

```bash
docker compose -f docker-compose-alt-ports.yml up -d
sleep 90
docker exec yb-single-alt /home/yugabyte/bin/ysqlsh -h localhost -p 5433 -U yugabyte -c "SELECT version();"
```

### 5. **Если порты заняты системными процессами** (не Docker)

Если `netstat` показывает, что порты заняты не Docker-контейнерами:

```bash
# Найдите PID процесса, занимающего порт 5433
sudo lsof -i :5433

# Если это системный PostgreSQL, остановите его (если можно)
sudo systemctl stop postgresql 2>/dev/null || true

# Или убейте процесс (если уверены, что он не нужен)
# sudo kill -9 <PID>
```

## 📊 Быстрая диагностика и решение за 5 минут

Выполните эту последовательность команд:

```bash
# 1. Очистите всё
docker stop $(docker ps -q) 2>/dev/null || true
docker rm $(docker ps -a -q) 2>/dev/null || true

# 2. Проверьте свободны ли порты
echo "=== Проверка портов ==="
for port in 5433 5434 9042 9043 7000 7001; do
  if sudo ss -tln | grep -q ":$port "; then
    echo "Порт $port ЗАНЯТ"
  else
    echo "Порт $port СВОБОДЕН"
  fi
done

# 3. Запустите с первым свободным портом для YSQL
docker run -d \
  --name yb-test-$(date +%s) \
  -p 5434:5433 \
  -p 9043:9042 \
  -p 7001:7000 \
  yugabytedb/yugabyte:latest \
  /home/yugabyte/bin/yugabyted start \
  --base_dir=/home/yugabyte/yb_data \
  --daemon=false

# 4. Проверьте
sleep 75
docker ps --format "table {{.Names}}\t{{.Ports}}\t{{.Status}}"
docker exec yb-test-$(docker ps -lq) /home/yugabyte/bin/ysqlsh -h localhost -p 5433 -U yugabyte -c "SELECT version();" 2>&1
```

## 🎯 Ключевые моменты

1. **Проблема**: Порт 5433 занят (скорее всего предыдущим контейнером).
2. **Решение 1**: Остановить все контейнеры и освободить порты.
3. **Решение 2**: Использовать другие порты (5434, 9043, 7001).
4. **Рекомендация**: Используйте **Решение 2** — это быстрее и не требует остановки возможных работающих сервисов.

**Попробуйте команду из пункта 3** (запуск с альтернативными портами). Если сработает — отлично, продолжаем с PostGIS. Если нет — посмотрим на вывод `netstat`/`ss`, чтобы понять, что именно занимает порты.



----------------------------------------



```
ничего не помогло, все то же самое
[]# docker exec yb-single-alt /home/yugabyte/bin/ysqlsh -h localhost -p 5433 -U yugabyte -c "SELECT version();"
ysqlsh: error: connection to server at "localhost" (::1), port 5433 failed: Connection refused
        Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 5433 failed: Connection refused
        Is the server running on that host and accepting TCP/IP connections?
[]#
[]#
[]#
[]# docker exec yb-single-alt /home/yugabyte/bin/ysqlsh -h localhost -p 5434 -U yugabyte -c "SELECT version();"
ysqlsh: error: connection to server at "localhost" (::1), port 5434 failed: Connection refused
        Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 5434 failed: Connection refused
        Is the server running on that host and accepting TCP/IP connections?
запуск одиночного контейнера не нашел пути, а я не знаю что это за пути и что нужно указать
docker run -d \
  --name yb-test \
  -p 5434:5433 \   # YSQL на порту 5434 вместо 5433
  -p 9043:9042 \   # YCQL на порту 9043 вместо 9042
  -p 7001:7000 \   # Web UI на порту 7001 вместо 7000
  yugabytedb/yugabyte:latest \
  /home/yugabyte/bin/yugabyted start \
  --base_dir=/home/yugabyte/yb_data \
  --daemon=false
docker: invalid reference format

Run 'docker run --help' for more information
bash: -p: command not found
bash: -p: command not found
bash: yugabytedb/yugabyte:latest: No such file or directory
```

Вероятно, в среде Docker возникла настройка сети или загрузка контейнера работает не до конца. Похожая ошибка с портами обсуждается в официальном форуме Yugabyte. Давайте это проверим.

### 🔍 Проверка состояния кластера и диагностика
Сначала нужно убедиться, что сам кластер и процессы внутри контейнера запущены и готовы.

```bash
# 1. Проверьте статус контейнера и потребление ресурсов
docker ps -a --filter "name=yb-single-alt"

# 2. Проверьте логи основного процесса yugabyted внутри контейнера
docker logs yb-single-alt --tail 50

# 3. Узнайте внутренний IP контейнера (важная информация)
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' yb-single-alt
```
### 🤔 Наиболее вероятные причины и быстрые проверки
*   **Процесс не запущен или завис.** Проверьте логи — если процесс не завершил инициализацию, вы увидите это.
*   **Неверный адрес для подключения.** Это частая проблема: контейнер может слушать на своем внутреннем IP (например, `172.17.0.2`), а не на `localhost`.
*   **Общая нестабильность тега `latest`.** В сообществе отмечают, что этот тег может быть проблемным; лучше использовать конкретный LTS-выпуск, например `2025.2.0.1-b1`.

### 🚀 Прямое решение: запуск по официальной инструкции
Лучший способ решить проблему — начать с рабочего конфига по документации. Запустите команду ниже, она использует стабильный тег и все необходимые порты. Если хотите проверить работу PostGIS, используйте это окружение.

```bash
# Остановите старый контейнер, чтобы освободить порты
docker stop yb-single-alt && docker rm yb-single-alt

# Запустите новый по официальной инструкции
docker run -d --name yugabyte \
    -p 7000:7000 -p 9000:9000 -p 15433:15433 -p 5433:5433 -p 9042:9042 \
    yugabytedb/yugabyte:2025.2.0.1-b1 \
    bin/yugabyted start --background=false

# Дайте контейнеру время на запуск (1-2 минуты)
sleep 120

# Проверьте статус кластера
docker exec -it yugabyte yugabyted status

# Попробуйте подключиться, указав внутренний IP контейнера
CONTAINER_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' yugabyte)
docker exec -it yugabyte /home/yugabyte/bin/ysqlsh -h $CONTAINER_IP -p 5433 -U yugabyte -c "SELECT version();"
```

### 💡 Ошибка с `docker run`
Ошибка `"docker: invalid reference format"` и `"bash: -p: command not found"` возникла из-за **неправильного копирования команды**. В прошлом сообщении комментарии `# YSQL на порту...` нарушили формат. Правильная команда — без комментариев:

```bash
docker run -d --name yb-test -p 5434:5433 -p 9043:9042 -p 7001:7000 yugabytedb/yugabyte:2025.2.0.1-b1 bin/yugabyted start --base_dir=/home/yugabyte/yb_data --daemon=false
```

### 📋 Что дальше?
Выполните шаги по проверке состояния и запуску по официальной инструкции. Если проблема (Connection refused) останется, пожалуйста, пришлите:
1.  **Вывод команды `docker logs yb-single-alt --tail 20`** (или нового контейнера).
2.  **Внутренний IP контейнера**, который вы получите из `docker inspect`.
Это поможет точно определить, в чем дело.
