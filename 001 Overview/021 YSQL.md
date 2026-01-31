Проблема с портом 5433 — он уже занят старым контейнером `yugabyte-yb-tserver-1`. Нужно его удалить и использовать другой порт для теста.

### 🔧 Быстрое решение

```bash
# 1. Останавливаем и удаляем ВСЁ
docker compose down -v --remove-orphans

# 2. Проверяем, что порт 5433 свободен
sudo lsof -i :5433

# 3. Если кто-то использует, завершаем процесс или меняем порт
# Изменим порт на 5434 в docker-compose.yml:
sed -i 's/"5433:5433"/"5434:5433"/' docker-compose.yml

# 4. Запускаем заново
docker compose up -d

# 5. Ждем 2 минуты для инициализации
sleep 120
```

### 📋 Обновленный docker-compose.yml с разными портами

Или создайте новый файл с правильными портами:

```yaml
version: '3.8'

services:
  yb-master:
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-master"
      - "--fs_data_dirs=/mnt/master"
      - "--master_addresses=yb-master:7100"
      - "--rpc_bind_addresses=yb-master:7100"
      - "--replication_factor=1"  # Для тестов на одной машине
    mem_limit: 1024m
    ports:
      - "7000:7000"

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
      - "5434:5433"  # Измененный порт!
    depends_on:
      - yb-master
    mem_limit: 1536m

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
    mem_limit: 1536m

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
    mem_limit: 1536m
```

### 🚀 Запуск и проверка

```bash
# 1. Очищаем всё
docker compose down -v --remove-orphans 2>/dev/null || true
docker ps -aq --filter "name=yugabyte" | xargs -r docker rm -f
docker volume prune -f

# 2. Создаем файл с новым портом
cat > docker-compose.yml << 'EOF'
# Вставьте содержимое выше с портом 5434
EOF

# 3. Запускаем
docker compose up -d

# 4. Ждем 3 минуты (важно!)
echo "Ждем инициализации кластера из 4 узлов..."
sleep 180

# 5. Проверяем статус
docker compose ps

# 6. Создаем таблицу транзакций
docker exec yugabyte-yb-tserver1-1 yb-admin --master_addresses yb-master:7100 create_transaction_table transactions_system

# 7. Подключаемся через новый порт
psql -h localhost -p 5434 -U yugabyte -c "CREATE TABLE test (id SERIAL PRIMARY KEY);"
psql -h localhost -p 5434 -U yugabyte -c "\dt"
```

### 💡 Ключевые моменты

1. **`--replication_factor=1`** в конфиге мастера — критически важно для работы с 1 TServer
2. **Порт 5434** вместо 5433, если 5433 занят
3. **3 TServer** для распределенного кластера (можно начать и с 1 TServer с `RF=1`)
4. **Долгое ожидание** (3 минуты) — кластеру из 4 узлов нужно время на выбор лидера и синхронизацию

### ✅ Альтернатива: самый простой рабочий вариант

Если хотите быстрее получить работающий кластер:

```bash
# Простой вариант с 1 мастером и 1 TServer, RF=1
docker run -d \
  --name yb-simple \
  -p 5435:5433 \
  -p 7001:7000 \
  --memory=3072m \
  yugabytedb/yugabyte:latest \
  bin/yugabyted start \
  --daemon=false \
  --ui=false \
  --listen=0.0.0.0 \
  --replication_factor=1

sleep 120
psql -h localhost -p 5435 -U yugabyte -c "CREATE TABLE simple (id SERIAL PRIMARY KEY);"
```

**Какой вариант предпочитаете:**
1. **Мультиузловой кластер** (4 узла) с портом 5434
2. **Простой вариант** (1 контейнер) с портом 5435 для быстрого старта

Выберите один и выполните команды.
