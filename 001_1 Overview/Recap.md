# Развертывание отказоустойчивого кластера YugabyteDB на CentOS 7

## 1. Предварительные проверки и настройка

### Проверка системы
```bash
# Версия ОС
cat /etc/centos-release

# Проверка свободных портов
sudo ss -tulpn | grep -E ':(5433|7000|9000)'
```

### Установка Docker и Docker Compose
```bash
# Удаляем при необходимости старые версии
sudo yum remove -y docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine

# Устанавливаем репозиторий
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Устанавливаем Docker
sudo yum install -y docker-ce docker-ce-cli containerd.io

# Устанавливаем Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Запускаем Docker
sudo systemctl start docker
sudo systemctl enable docker

# Проверка
docker --version
docker compose --version
```

### Настройка пользователя
```bash
# Добавляем пользователя в группу docker
sudo usermod -aG docker $USER
# НЕОБХОДИМО выйти и зайти заново или выполнить:
newgrp docker

# Проверяем
id $USER | grep docker
```

---

## 2. Установка и запуск кластера

### Основные ошибки:

1. **Недостаток памяти** — YugabyteDB требует минимум 2-3 ГБ на узел
2. **Конфликт портов** — 5433, 7000, 9000
3. **Неправильный replication_factor** — RF=3 требует 3 TServer
4. **Слишком быстрый запуск** — нужно ждать инициализации системных таблиц

### Создание рабочей конфигурации

Создаем директорию и файл `docker-compose.yml`:

```bash
mkdir ~/yugabyte-cluster && cd ~/yugabyte-cluster
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Главный узел управления
  yb-master:
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-master"
      - "--fs_data_dirs=/mnt/master"
      - "--master_addresses=yb-master:7100"
      - "--rpc_bind_addresses=yb-master:7100"
      - "--replication_factor=1"  # КРИТИЧЕСКИ ВАЖНО для локального теста!
    mem_limit: 1024m
    ports:
      - "7000:7000"  # Веб-интерфейс

  # Три узла данных (TServer)
  yb-tserver1:
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-tserver"
      - "--fs_data_dirs=/mnt/tserver"
      - "--tserver_master_addrs=yb-master:7100"
      - "--rpc_bind_addresses=yb-tserver1:9100"
      - "--enable_ysql"
      - "--ysql_enable_auth=false"
      - "--pgsql_proxy_bind_address=0.0.0.0:5433"
    ports:
      - "5433:5433"  # PostgreSQL-совместимый порт
    depends_on:
      - yb-master
    mem_limit: 2048m  # Минимум 2 ГБ!

  yb-tserver2:
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-tserver"
      - "--fs_data_dirs=/mnt/tserver"
      - "--tserver_master_addrs=yb-master:7100"
      - "--rpc_bind_addresses=yb-tserver2:9100"
      - "--enable_ysql"
      - "--ysql_enable_auth=false"
    depends_on:
      - yb-master
    mem_limit: 2048m

  yb-tserver3:
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-tserver"
      - "--fs_data_dirs=/mnt/tserver"
      - "--tserver_master_addrs=yb-master:7100"
      - "--rpc_bind_addresses=yb-tserver3:9100"
      - "--enable_ysql"
      - "--ysql_enable_auth=false"
    depends_on:
      - yb-master
    mem_limit: 2048m
```

### Пошаговый запуск

```bash
# 1. Останавливаем и удаляем все предыдущие контейнеры
docker-compose down -v 2>/dev/null || true
docker ps -aq --filter "name=yugabyte" | xargs -r docker rm -f
docker volume prune -f

# 2. Запускаем Master и ждем его готовности
docker-compose up -d yb-master
echo "Ждем запуска мастера (30 секунд)..."
sleep 30

# 3. Запускаем все TServer
docker-compose up -d
echo "Ждем полной инициализации кластера (2 минуты)..."
sleep 120

# 4. Проверка запуска
docker-compose ps
```

### Критическая проверка: системные таблицы транзакций

```bash
# Проверяем, создалась ли таблица транзакций
docker exec yugabyte-yb-master-1 yb-admin --master_addresses yb-master:7100 list_tables | grep -i trans

# Если таблицы нет, создаем вручную (для RF=1)
docker exec yugabyte-yb-tserver1-1 yb-admin --master_addresses yb-master:7100 create_transaction_table transactions_system

# Ждем еще 30 секунд
sleep 30
```

### Финальная проверка работоспособности

```bash
# Установка PostgreSQL клиента (если нет)
sudo yum install -y postgresql

# Тест подключения
psql -h localhost -p 5433 -U yugabyte -c "SELECT version();"

# Создание тестовой таблицы (должно работать!)
psql -h localhost -p 5433 -U yugabyte -c "
CREATE TABLE test_success (
    id SERIAL PRIMARY KEY,
    message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO test_success (message) VALUES ('Кластер успешно запущен!');

SELECT * FROM test_success;
"

# Проверка в веб-интерфейсе
echo "Веб-интерфейс доступен по адресу: http://$(hostname -I | awk '{print $1}'):7000"
```

---

## 3. Тестирование и важные особенности

### Тестирование отказоустойчивости (с ограничениями)

```bash
# !!! у нас проброшен порт 5433 только на yb-tserver1
# Поэтому при его остановке внешние подключения прекратятся!

# 1. Останавливаем НЕ основной TServer (не yb-tserver1!)
docker-compose stop yb-tserver2

# 2. Проверяем, что кластер продолжает работать внутри
docker exec yugabyte-yb-tserver1-1 ysqlsh -U yugabyte -c "SELECT * FROM test_success;"

# 3. Восстанавливаем
docker-compose start yb-tserver2
sleep 30

# 4. Теперь пробуем остановить yb-tserver1 (основной)
docker-compose stop yb-tserver1

# 5. Внешние подключения НЕ РАБОТАЮТ (порт 5433 недоступен)
psql -h localhost -p 5433 -U yugabyte -c "SELECT 1;" || echo "Ожидаемо: подключение разорвано"

# 6. Но кластер внутри работает!
docker exec yugabyte-yb-tserver2-1 ysqlsh -U yugabyte -h yb-tserver2 -c "SELECT * FROM test_success;"

# 7. Восстанавливаем
docker-compose start yb-tserver1
sleep 30
```

### Решение проблемы единой точки входа

**Вариант 1: Пробросить порты у всех TServer**

```yaml
# В docker-compose.yml для каждого TServer добавляем:
yb-tserver1:
  ports:
    - "5433:5433"
    
yb-tserver2:
  ports:
    - "5434:5433"  # Другой порт
    
yb-tserver3:
  ports:
    - "5435:5433"  # Еще один порт
```

**Вариант 2: Использовать Smart Driver (рекомендуется для production)**

Smart Driver автоматически подключается к любому живому TServer.

---

## 4. Архитектура и матчасть

### Вопросы с ответами:

**Q1: Почему возникают проблемы с созданием системных таблиц?**  
**A:** Системная таблица `transactions_system` может не инициализироваться автоматически из-за:
- Недостатка памяти
- `replication_factor=3` при только 1 TServer
- Слишком быстрого запуска без ожидания

**Q2: Почему минимум 3 TServer?**  
**A:** По умолчанию `replication_factor=3` (данные в 3 копиях). Для отказоустойчивости нужно `N+2` узлов, где N — количество отказов, которые нужно пережить.

**Q3: Как данные распределяются между узлами?**  
**A:** Автоматически через шардирование (разбиение на таблеты). Каждый таблет реплицируется на RF узлов.

### Краткая архитектурная схема:

```
Приложение
    ↓
PostgreSQL-драйвер (порт 5433)
    ↓
YugabyteDB TServer (x3)
    ├─ YSQL API (PostgreSQL-совместимость)
    ├─ YCQL API (Cassandra-совместимость)  
    └─ DocDB (распределенное хранилище)
        ↓
    RAFT консенсус (репликация)
        ↓
Физические узлы с данными
```

### Ключевые конфигурационные параметры:

| Параметр | Значение | Объяснение |
|----------|----------|------------|
| `--replication_factor` | 1 (тест), 3 (prod) | Количество копий данных |
| `--enable_ysql` | true | Включает PostgreSQL-совместимость |
| `mem_limit` | 2048m+ | Минимум 2 ГБ на TServer |
| Порт 5433 | yb-tserver1 | Точка входа для PostgreSQL |

---

## Быстрый старт

### Минимальная рабочая конфигурация:
```bash
sudo yum install -y docker postgresql
sudo systemctl start docker
sudo usermod -aG docker $USER && newgrp docker
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

mkdir ~/yugabyte && cd ~/yugabyte
# Создаем docker-compose.yml
docker-compose up -d
sleep 150
psql -h localhost -p 5433 -U yugabyte -c "CREATE TABLE test(id SERIAL PRIMARY KEY); SELECT 'SUCCESS' as status;"
```
