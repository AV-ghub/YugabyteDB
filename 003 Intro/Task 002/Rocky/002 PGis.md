```
вот так вот

docker ps
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
[root@pg138 yugabyte]# sudo systemctl start docker
Job for docker.service failed because the control process exited with error code.
See "systemctl status docker.service" and "journalctl -xeu docker.service" for details.

sudo journalctl -u docker --no-pager -n 50
Jan 25 10:16:16  systemd[1]: Starting Docker Application Container Engine...
Jan 25 10:16:17  dockerd[1047]: unable to configure the Docker daemon with file /etc/docker/daemon.json: the following directives are specified both as a flag and in the configuration file: hosts: (from flag: [fd://], from file: [tcp://0.0.0.0:2375 unix:///var/run/docker.sock])
Jan 25 10:16:17  systemd[1]: docker.service: Main process exited, code=exited, status=1/FAILURE
Jan 25 10:16:17  systemd[1]: docker.service: Failed with result 'exit-code'.
Jan 25 10:16:17  systemd[1]: Failed to start Docker Application Container Engine.
Jan 25 10:16:19  systemd[1]: docker.service: Scheduled restart job, restart counter is at 1.
Jan 25 10:16:19  systemd[1]: Stopped Docker Application Container Engine.
Jan 25 10:16:19  systemd[1]: Starting Docker Application Container Engine...
Jan 25 10:16:19  dockerd[1511]: unable to configure the Docker daemon with file /etc/docker/daemon.json: the following directives are specified both as a flag and in the configuration file: hosts: (from flag: [fd://], from file: [tcp://0.0.0.0:2375 unix:///var/run/docker.sock])
Jan 25 10:16:19  systemd[1]: docker.service: Main process exited, code=exited, status=1/FAILURE
Jan 25 10:16:19  systemd[1]: docker.service: Failed with result 'exit-code'.
Jan 25 10:16:19  systemd[1]: Failed to start Docker Application Container Engine.

```

Отлично! Проблема найдена. Ошибка говорит о **конфликте конфигурации**: параметр `hosts` указан и в файле `/etc/docker/daemon.json`, и в аргументах запуска systemd. Это частая проблема после миграции или обновления.

## 🔧 Решение: Исправление конфликта конфигурации

### Шаг 1: Проверьте и исправьте `/etc/docker/daemon.json`

```bash
# Посмотрите текущий файл конфигурации
sudo cat /etc/docker/daemon.json

# Скорее всего, там есть строка с "hosts". Отредактируйте файл:
sudo nano /etc/docker/daemon.json
```

**Ожидаемое содержимое файла** (без параметра `hosts`):
```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
```

Если у вас в файле есть строка `"hosts": ["tcp://0.0.0.0:2375", "unix:///var/run/docker.sock"]` — **удалите её или закомментируйте**.

### Шаг 2: Проверьте аргументы systemd для Docker

```bash
# Посмотрите, как запускается Docker через systemd
sudo systemctl cat docker | grep ExecStart

# Если видите что-то вроде: ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
# То это нормально. Проблема именно в дублировании hosts в daemon.json
```

### Шаг 3: Перезапустите Docker

```bash
# Перезагрузите конфигурацию systemd
sudo systemctl daemon-reload

# Запустите Docker
sudo systemctl start docker

# Проверьте статус
sudo systemctl status docker

# Включите автозапуск
sudo systemctl enable docker
```

### Шаг 4: Если проблема сохраняется — временное решение

Если не удаётся быстро исправить конфликт, можно **временно удалить файл конфигурации**:

```bash
# Создайте резервную копию
sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.backup

# Удалите файл (или переместите)
sudo mv /etc/docker/daemon.json /etc/docker/daemon.json.disabled

# Перезапустите Docker
sudo systemctl daemon-reload
sudo systemctl start docker
```

## 📝 Альтернативный подход: Настройка systemd без daemon.json

Если хотите настроить Docker для прослушивания TCP-порта (например, для удалённого доступа), сделайте это через systemd:

```bash
# Создайте или отредактируйте drop-in файл для Docker
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/override.conf << 'EOF'
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2375
EOF

# Перезагрузите конфигурацию
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 🚀 Краткая последовательность команд для быстрого восстановления

Выполните эти команды по порядку:

```bash
# 1. Создайте резервную копию текущей конфигурации
sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.backup 2>/dev/null || true

# 2. Установите минимальную рабочую конфигурацию
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

# 3. Перезагрузите systemd и перезапустите Docker
sudo systemctl daemon-reload
sudo systemctl restart docker

# 4. Проверьте статус
sudo systemctl status docker --no-pager -l
docker run --rm hello-world
```

## 🔍 Диагностика, если Docker всё равно не запускается

Если после этих действий Docker не запускается:

```bash
# Посмотрите полные логи
sudo journalctl -u docker --no-pager -n 100

# Проверьте, нет ли других конфигурационных файлов
ls -la /etc/docker/daemon.json*
ls -la /etc/systemd/system/docker.service.d/

# Попробуйте запустить dockerd вручную для диагностики
sudo /usr/bin/dockerd --debug
```

## 💡 Причина проблемы

Конфликт возникает, когда:
1. В `/etc/docker/daemon.json` указан параметр `hosts`
2. В то же время systemd запускает Docker с флагом `-H fd://` (по умолчанию)
3. Docker не может определить, какую конфигурацию использовать

**Решение:** Либо удалите `hosts` из `daemon.json`, либо настройте systemd не передавать флаг `-H`.

Попробуйте минимальную конфигурацию (шаг 2 в краткой последовательности). Если сработает — Docker запустится, и мы сможем продолжить с YugabyteDB и PostGIS.
