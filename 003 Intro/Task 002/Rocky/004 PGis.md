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
mkdir ~/yugabyte-test
cd ~/yugabyte-test

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
