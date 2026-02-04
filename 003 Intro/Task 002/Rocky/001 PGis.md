docker --version
Docker version 29.0.2, build 8108357

docker compose version
Docker Compose version v2.40.3

при этом

docker ps
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?

docker compose ps
no configuration file provided: not found

```

Отлично, вы мигрировали на современную систему! Проблема с Docker на Rocky Linux 9 обычно решается за несколько шагов. Судя по ошибке, Docker Daemon не запущен или у вас нет прав на его использование.

## 🚀 Восстановление работы Docker на Rocky Linux 9

Выполните эти команды **последовательно**:

### 1. **Запуск и включение Docker Daemon**

```bash
# Проверьте статус службы Docker
sudo systemctl status docker

# Если служба неактивна, запустите её
sudo systemctl start docker

# Включите автозапуск Docker при загрузке системы
sudo systemctl enable docker

# Проверьте, что служба работает
sudo systemctl status docker | grep "Active:"
```

### 2. **Добавление пользователя в группу `docker`** (чтобы не использовать `sudo`)

```bash
# Добавьте текущего пользователя в группу docker
sudo usermod -aG docker $USER

# Для применения изменений нужно выйти и зайти в систему заново
# Если не хотите перезаходить, используйте:
newgrp docker

# Проверьте, что вы в группе docker
groups
```

### 3. **Проверка работоспособности Docker**

```bash
# Теперь проверьте без sudo
docker ps
docker run hello-world
```

### 4. **Настройка Docker Compose** (если потребуется)

У вас уже установлен Docker Compose v2 (часть Docker Desktop). Для работы с ним нужно использовать команду `docker compose` (через пробел):

```bash
# Проверьте версию
docker compose version

# Создайте тестовый docker-compose.yml
cat > docker-compose.test.yml << 'EOF'
version: '3.8'
services:
  test:
    image: hello-world
    container_name: test-container
EOF

# Запустите тестовый compose
docker compose -f docker-compose.test.yml up
```

## 📋 Полная последовательность команд для восстановления

Вот единый блок команд для копирования и выполнения:

```bash
# 1. Запуск Docker Daemon
sudo systemctl start docker
sudo systemctl enable docker

# 2. Добавление пользователя в группу docker
sudo usermod -aG docker $USER
newgrp docker

# 3. Проверка
docker ps
docker run --rm hello-world

# 4. Создание тестового docker-compose.yml для YugabyteDB
cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  yb-master:
    image: yugabytedb/yugabyte:latest
    container_name: yb-master
    command: [ "/home/yugabyte/bin/yb-master",
               "--fs_data_dirs=/mnt/data",
               "--master_addresses=yb-master:7100",
               "--rpc_bind_addresses=yb-master:7100",
               "--enable_ysql=true",
               "--callhome_enabled=false" ]
    ports:
      - "7000:7000"
    volumes:
      - yb-master-data:/mnt/data
    networks:
      - yb-network

  yb-tserver:
    image: yugabytedb/yugabyte:latest
    container_name: yb-tserver
    command: [ "/home/yugabyte/bin/yb-tserver",
               "--fs_data_dirs=/mnt/data",
               "--tserver_master_addrs=yb-master:7100",
               "--rpc_bind_addresses=yb-tserver:9100",
               "--enable_ysql=true",
               "--callhome_enabled=false" ]
    ports:
      - "5433:5433"
      - "9042:9042"
      - "9000:9000"
    volumes:
      - yb-tserver-data:/mnt/data
    depends_on:
      - yb-master
    networks:
      - yb-network

networks:
  yb-network:
    driver: bridge

volumes:
  yb-master-data:
  yb-tserver-data:
EOF

# 5. Запуск кластера YugabyteDB
docker compose up -d

# 6. Проверка статуса
docker compose ps
sleep 30  # Дайте время на запуск
docker exec yb-tserver /home/yugabyte/bin/ysqlsh -h localhost -p 5433 -U yugabyte -c "SELECT version();"
```

## 🔧 Если Docker всё равно не запускается

Если после этих шагов Docker Daemon не запускается, проверьте:

```bash
# 1. Просмотр логов Docker
sudo journalctl -u docker --no-pager -n 50

# 2. Проверка наличия конфликтующих служб (podman, containerd)
sudo systemctl status podman containerd

# 3. Полная переустановка Docker (если нужно)
sudo dnf remove -y docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl start docker
```

## 🎯 Краткий чек-лист

1. **✅ Запущен ли Docker Daemon?** → `sudo systemctl status docker`
2. **✅ Вы в группе docker?** → `groups | grep docker`
3. **✅ Работает ли hello-world?** → `docker run hello-world`
4. **✅ Используете правильную команду compose?** → `docker compose` (не `docker-compose`)

Попробуйте выполнить шаги по порядку и сообщите, на каком этапе возникнут проблемы. После восстановления Docker мы сможем продолжить с PostGIS на чистой системе.
