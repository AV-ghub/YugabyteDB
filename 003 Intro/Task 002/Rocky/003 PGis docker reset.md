```
вот так
sudo systemctl status docker
× docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: disabled)
     Active: failed (Result: exit-code) since Wed 2026-02-04 16:05:27 UTC; 11s ago
TriggeredBy: × docker.socket
       Docs: https://docs.docker.com
    Process: 168296 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock (code=exited, status=1/FAILURE)
   Main PID: 168296 (code=exited, status=1/FAILURE)
        CPU: 62ms

Feb 04 16:05:27  systemd[1]: docker.service: Scheduled restart job, restart counter is at 3.
Feb 04 16:05:27  systemd[1]: Stopped Docker Application Container Engine.
Feb 04 16:05:27  systemd[1]: docker.service: Start request repeated too quickly.
Feb 04 16:05:27  systemd[1]: docker.service: Failed with result 'exit-code'.
Feb 04 16:05:27  systemd[1]: Failed to start Docker Application Container Engine.

sudo journalctl -u docker --no-pager -n 50
Jan 25 10:16:24 systemd[1]: docker.service: Failed with result 'exit-code'.
Jan 25 10:16:24 systemd[1]: Failed to start Docker Application Container Engine.
Feb 04 15:59:02 systemd[1]: Starting Docker Application Container Engine...
Feb 04 15:59:02 dockerd[168110]: unable to configure the Docker daemon with file /etc/docker/daemon.json: the following directives are specified both as a flag and in the configuration file: hosts: (from flag: [fd://], from file: [tcp://0.0.0.0:2375 unix:///var/run/docker.sock])
Feb 04 15:59:02 systemd[1]: docker.service: Main process exited, code=exited, status=1/FAILURE
Feb 04 15:59:02 systemd[1]: docker.service: Failed with result 'exit-code'.
Feb 04 15:59:02 systemd[1]: Failed to start Docker Application Container Engine.
Feb 04 15:59:04 systemd[1]: docker.service: Scheduled restart job, restart counter is at 1.
Feb 04 15:59:04 systemd[1]: Stopped Docker Application Container Engine.
Feb 04 15:59:04 systemd[1]: Starting Docker Application Container Engine...
Feb 04 15:59:04 dockerd[168125]: unable to configure the Docker daemon with file /etc/docker/daemon.json: the following directives are specified both as a flag and in the configuration file: hosts: (from flag: [fd://], from file: [tcp://0.0.0.0:2375 unix:///var/run/docker.sock])
Feb 04 15:59:04 systemd[1]: docker.service: Main process exited, code=exited, status=1/FAILURE
Feb 04 15:59:04 systemd[1]: docker.service: Failed with result 'exit-code'.
Feb 04 15:59:04 systemd[1]: Failed to start Docker Application Container Engine.
Feb 04 15:59:06 systemd[1]: docker.service: Scheduled restart job, restart counter is at 2.
```

Проблема сохраняется — значит, в `/etc/docker/daemon.json` всё ещё есть параметр `hosts`. Давайте **полностью пересоздадим конфигурацию Docker** на чистом листе.

## 🎯 Полное решение: Сброс конфигурации Docker

Выполните эти команды по порядку:

### 1. **Полностью удалите старую конфигурацию**

```bash
# 1. Остановите Docker (если он пытается запускаться)
sudo systemctl stop docker docker.socket containerd 2>/dev/null || true

# 2. Удалите ВСЕ конфигурационные файлы Docker
sudo rm -f /etc/docker/daemon.json
sudo rm -f /etc/docker/daemon.json.backup
sudo rm -f /etc/docker/daemon.json.disabled

# 3. Удалите drop-in файлы systemd для Docker
sudo rm -rf /etc/systemd/system/docker.service.d/

# 4. Сбросьте конфигурацию systemd
sudo systemctl daemon-reload
sudo systemctl reset-failed docker docker.socket
```

### 2. **Проверьте, что файл daemon.json действительно удалён**

```bash
# Посмотрите, есть ли какие-либо файлы конфигурации Docker
ls -la /etc/docker/ 2>/dev/null || echo "Директория /etc/docker/ не существует"

# Если директория не существует — создайте её
sudo mkdir -p /etc/docker
```

### 3. **Создайте минимальную рабочую конфигурацию**

```bash
# Создайте чистый daemon.json с минимальными настройками
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
EOF

# Установите правильные права
sudo chmod 644 /etc/docker/daemon.json
```

### 4. **Перезапустите Docker с чистого листа**

```bash
# 1. Перезагрузите systemd
sudo systemctl daemon-reload

# 2. Запустите Docker
sudo systemctl start docker

# 3. Проверьте статус
sudo systemctl status docker --no-pager

# 4. Если статус показывает ошибку, посмотрите конкретную причину
sudo journalctl -u docker --no-pager -n 20
```

### 5. **Если Docker всё равно не запускается — диагностика**

```bash
# Попробуйте запустить dockerd вручную с выводом логов
sudo /usr/bin/dockerd --debug 2>&1 | head -50

# Проверьте, нет ли других служб, занимающих порты
sudo netstat -tlnp | grep -E ':(2375|2376)'

# Проверьте SELinux (частая проблема на Rocky Linux)
sudo sestatus
sudo getenforce
```

## 🚨 Экстренное решение: Переустановка Docker

Если ничего не помогает, полностью переустановите Docker:

```bash
# 1. Удалите Docker
sudo dnf remove -y docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine docker-ce docker-ce-cli containerd.io

# 2. Удалите все конфигурационные файлы
sudo rm -rf /var/lib/docker
sudo rm -rf /etc/docker
sudo rm -rf /etc/systemd/system/docker.service.d

# 3. Установите Docker заново
sudo dnf install -y yum-utils
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 4. Запустите Docker
sudo systemctl start docker
sudo systemctl enable docker

# 5. Проверьте
docker run --rm hello-world
```

## 📊 Краткий чек-лист восстановления

1. ✅ **Удалены все конфигурационные файлы** (`/etc/docker/daemon.json*`, `/etc/systemd/system/docker.service.d/`)
2. ✅ **Создан чистый daemon.json** без параметра `hosts`
3. ✅ **Systemd перезагружен** (`systemctl daemon-reload`)
4. ✅ **Docker запускается** (`systemctl start docker`)
5. ✅ **Проверен hello-world** (`docker run hello-world`)

## 🔧 Что делать дальше

После успешного запуска Docker:

```bash
# 1. Добавьте пользователя в группу docker
sudo usermod -aG docker $USER
newgrp docker

# 2. Скачайте образ YugabyteDB
docker pull yugabytedb/yugabyte:latest

# 3. Создайте тестовый docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  yb-test:
    image: yugabytedb/yugabyte:latest
    container_name: yb-test-single
    command: ["/home/yugabyte/bin/yugabyted", "start", "--background=false"]
    ports:
      - "5433:5433"
      - "9042:9042"
      - "7000:7000"
    volumes:
      - yb-data:/home/yugabyte/yb_data
volumes:
  yb-data:
EOF

# 4. Запустите тестовый кластер
docker compose up -d
sleep 30
docker exec yb-test-single /home/yugabyte/bin/ysqlsh -c "SELECT version();"
```

Попробуйте **шаг 1-4 полного решения** (сброс конфигурации). Если Docker запустится — отлично! Если нет — пришлите вывод команд `sudo journalctl -u docker --no-pager -n 20` и `sudo /usr/bin/dockerd --debug 2>&1 | head -50`.
