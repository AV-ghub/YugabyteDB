**Отлично! Теперь мы нашли коренную причину!** Ошибка говорит, что для создания таблицы с `replication_factor=3` (репликация в 3 копии) нужно **минимум 2 TServer**, а у вас только 1.

### 🔧 Решение: Создаем кластер с правильным replication_factor

Ваш текущий docker-compose.yml создает кластер с 1 TServer, но `replication_factor=3` по умолчанию. Нужно либо добавить TServer-ы, либо изменить `replication_factor`.

**Вариант 1: Добавляем еще 2 TServer (рекомендуется для распределенного кластера)**

Создайте новый `docker-compose.yml`:

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
      - "5433:5433"
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

**Вариант 2: Изменяем replication_factor на 1 (для тестов на одной машине)**

Выйдите из контейнера (`exit`) и выполните:

```bash
# 1. Останавливаем текущий кластер
docker compose down -v

# 2. Создаем новый docker-compose.yml с replication_factor=1
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  yb-master:
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-master"
      - "--fs_data_dirs=/mnt/master"
      - "--master_addresses=yb-master:7100"
      - "--rpc_bind_addresses=yb-master:7100"
      - "--replication_factor=1"  # Ключевое изменение!
    mem_limit: 1024m
    ports:
      - "7000:7000"

  yb-tserver:
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-tserver"
      - "--fs_data_dirs=/mnt/tserver"
      - "--tserver_master_addrs=yb-master:7100"
      - "--rpc_bind_addresses=yb-tserver:9100"
      - "--enable_ysql"
      - "--ysql_enable_auth=false"
      - "--pgsql_proxy_bind_address=0.0.0.0:5433"
    ports:
      - "5433:5433"
    depends_on:
      - yb-master
    mem_limit: 2048m
EOF

# 3. Запускаем
docker compose up -d

# 4. Ждем 2 минуты
sleep 120

# 5. Создаем таблицу транзакций (теперь должна работать)
docker exec yugabyte-yb-tserver-1 yb-admin --master_addresses yb-master:7100 create_transaction_table transactions_system

# 6. Пробуем создать таблицу
psql -h localhost -p 5433 -U yugabyte -c "CREATE TABLE test (id SERIAL PRIMARY KEY);"
```

### 📊 Почему это важно

**Replication Factor (RF)** определяет, в сколько копий реплицируются данные:
- **RF=3** (по умолчанию): нужно минимум 3 TServer для отказоустойчивости
- **RF=1**: достаточно 1 TServer, но нет отказоустойчивости

Для **локального тестирования** используйте `RF=1`. Для **распределенного кластера** нужны 3+ TServer с `RF=3`.

### 🎯 Рекомендация

**Используйте Вариант 2** (с `replication_factor=1`) для быстрого старта и обучения. Позже, когда освоите основы, можно перейти к распределенному кластеру.

**Выберите вариант и выполните команды.** Сообщите о результате, особенно шага 5 и 6 в Варианте 2.
