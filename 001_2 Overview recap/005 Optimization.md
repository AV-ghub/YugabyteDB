```
вот при такой раскладке
# free
              total        used        free      shared  buff/cache   available
Mem:      131863192     3886548   116133016     2739956    11843628   124159668
Swap:       4194300           0     4194300
какие изменения в нашем рабочем конфиге порекомендуешь сделать?
```
Отличный вопрос! С такими ресурсами (126 ГБ RAM) вы можете **значительно улучшить производительность** кластера. Давайте оптимизируем конфиг под ваш мощный стенд.

## 🚀 Оптимизированный конфиг для мощного сервера

```yaml
version: '3.8'

services:
  yb-master:
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-master"
      - "--fs_data_dirs=/mnt/master"
      - "--master_addresses=yb-master:7100,yb-master-b:7100,yb-master-c:7100"  # 3 мастера!
      - "--rpc_bind_addresses=yb-master:7100"
      - "--replication_factor=3"
      - "--default_memory_limit_to_ram_ratio=0.1"  # Лимит памяти 10% от RAM
    mem_limit: 4096m  # 4 ГБ для мастера
    volumes:
      - yb_master_data:/mnt/master
    ports:
      - "7000:7000"

  yb-master-b:  # Второй мастер для отказоустойчивости
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-master"
      - "--fs_data_dirs=/mnt/master"
      - "--master_addresses=yb-master:7100,yb-master-b:7100,yb-master-c:7100"
      - "--rpc_bind_addresses=yb-master-b:7100"
      - "--replication_factor=3"
      - "--default_memory_limit_to_ram_ratio=0.1"
    mem_limit: 4096m
    volumes:
      - yb_master_b_data:/mnt/master

  yb-master-c:  # Третий мастер
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-master"
      - "--fs_data_dirs=/mnt/master"
      - "--master_addresses=yb-master:7100,yb-master-b:7100,yb-master-c:7100"
      - "--rpc_bind_addresses=yb-master-c:7100"
      - "--replication_factor=3"
      - "--default_memory_limit_to_ram_ratio=0.1"
    mem_limit: 4096m
    volumes:
      - yb_master_c_data:/mnt/master

  yb-tserver-a:
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-tserver"
      - "--fs_data_dirs=/mnt/tserver"
      - "--tserver_master_addrs=yb-master:7100,yb-master-b:7100,yb-master-c:7100"
      - "--rpc_bind_addresses=yb-tserver-a:9100"
      - "--enable_ysql"
      - "--ysql_enable_auth=false"
      - "--pgsql_proxy_bind_address=0.0.0.0:5433"
      - "--memory_limit_hard_bytes=34359738368"  # 32 ГБ на TServer!
      - "--read_buffer_memory_limit=8589934592"   # 8 ГБ для read buffer
      - "--db_block_cache_size_percentage=30"     # 30% от memory_limit
      - "--ysql_num_shards_per_tserver=8"         # Больше шардов для параллелизма
      - "--yb_num_shards_per_tserver=8"
    ports:
      - "5433:5433"
    mem_limit: 36864m  # 36 ГБ (32 + буфер)
    volumes:
      - yb_tserver_a_data:/mnt/tserver

  yb-tserver-b:
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-tserver"
      - "--fs_data_dirs=/mnt/tserver"
      - "--tserver_master_addrs=yb-master:7100,yb-master-b:7100,yb-master-c:7100"
      - "--rpc_bind_addresses=yb-tserver-b:9100"
      - "--enable_ysql"
      - "--ysql_enable_auth=false"
      - "--pgsql_proxy_bind_address=0.0.0.0:5433"
      - "--memory_limit_hard_bytes=34359738368"
      - "--read_buffer_memory_limit=8589934592"
      - "--db_block_cache_size_percentage=30"
      - "--ysql_num_shards_per_tserver=8"
      - "--yb_num_shards_per_tserver=8"
    ports:
      - "5434:5433"
    mem_limit: 36864m
    volumes:
      - yb_tserver_b_data:/mnt/tserver

  yb-tserver-c:
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-tserver"
      - "--fs_data_dirs=/mnt/tserver"
      - "--tserver_master_addrs=yb-master:7100,yb-master-b:7100,yb-master-c:7100"
      - "--rpc_bind_addresses=yb-tserver-c:9100"
      - "--enable_ysql"
      - "--ysql_enable_auth=false"
      - "--pgsql_proxy_bind_address=0.0.0.0:5433"
      - "--memory_limit_hard_bytes=34359738368"
      - "--read_buffer_memory_limit=8589934592"
      - "--db_block_cache_size_percentage=30"
      - "--ysql_num_shards_per_tserver=8"
      - "--yb_num_shards_per_tserver=8"
    ports:
      - "5435:5433"
    mem_limit: 36864m
    volumes:
      - yb_tserver_c_data:/mnt/tserver

volumes:
  yb_master_data:
  yb_master_b_data:
  yb_master_c_data:
  yb_tserver_a_data:
  yb_tserver_b_data:
  yb_tserver_c_data:
```

## 📊 Что изменилось и почему

### 1. **Три мастера вместо одного** 🏛️
```yaml
master_addresses: yb-master:7100,yb-master-b:7100,yb-master-c:7100
```
- **Было**: 1 мастер = единая точка отказа
- **Стало**: 3 мастера = полноценный кворум
- **Результат**: Отказоустойчивость на уровне управления кластером

### 2. **Увеличенные лимиты памяти** 💾
- **TServer**: с 2 ГБ → до **32 ГБ рабочей памяти** + 4 ГБ буфер
- **Master**: с 1 ГБ → до **4 ГБ**
- **Обоснование**: 126 ГБ RAM позволяют выделить ~80 ГБ на YugabyteDB без ущерба системе

### 3. **Оптимизация кэшей** 🔥
```yaml
--db_block_cache_size_percentage=30     # 30% от memory_limit = ~9.6 ГБ кэша
--read_buffer_memory_limit=8589934592   # 8 ГБ буфера для операций чтения
```
Чем больше данных в памяти — тем меньше обращений к диску

### 4. **Увеличение числа шардов** ⚙️
```yaml
--ysql_num_shards_per_tserver=8  # Было 2 (по умолчанию)
```
- Больше параллелизма при обработке запросов
- Лучшее распределение нагрузки
- Особенно важно для аналитических запросов

## 💡 Дополнительные рекомендации для вашего стенда

### A. **Использование SSD/NVMe для томов**
Если у вас быстрые диски, подключите их:

```bash
# 1. Создайте директории на SSD
sudo mkdir -p /mnt/ssd/yugabyte/{master,tserver-a,tserver-b,tserver-c}

# 2. В docker-compose.yml замените блок volumes на:
volumes:
  yb_master_data:
    driver_opts:
      type: none
      o: bind
      device: /mnt/ssd/yugabyte/master
  # ... аналогично для остальных
```

### B. **Оптимизация для векторного поиска** (если будете тестировать AI/ML)
```yaml
--enable_automatic_tablet_splitting=false  # Отключаем авто-сплит для векторов
--maintenance_manager_num_threads=4        # Больше потоков для обслуживания
--raft_heartbeat_interval_ms=500           # Более частый heartbeat
```

### C. **Мониторинг и отладка**
```yaml
--ysql_log_statement=all        # Логировать все SQL запросы
--ysql_log_min_duration_statement=1000  # Логировать медленные (>1сек)
--metric_node_name=yb-tserver-a  # Уникальное имя для метрик
```

## 🧪 Тест производительности после оптимизации

```bash
# Создайте тестовую нагрузку
psql -h localhost -p 5433 -U yugabyte << 'EOF'
-- Создаем большую таблицу
CREATE TABLE perf_test AS
SELECT 
  generate_series(1, 10000000) as id,
  md5(random()::text) as data,
  (random() * 1000)::int as category;

-- Тест агрегаций
EXPLAIN (ANALYZE, BUFFERS) 
SELECT category, COUNT(*), AVG(id)
FROM perf_test 
GROUP BY category 
ORDER BY COUNT(*) DESC 
LIMIT 10;

-- Проверяем использование кэша
SELECT * FROM yb_servers();
EOF
```

## 📈 Ожидаемые улучшения

| Аспект | Было (старый конфиг) | Стало (новый конфиг) | Улучшение |
|--------|---------------------|---------------------|-----------|
| **Память на TServer** | 2 ГБ | 32 ГБ | 16x |
| **Кэш данных** | ~0.5 ГБ | ~9.6 ГБ | 19x |
| **Параллелизм** | 2 шарда/узел | 8 шардов/узел | 4x |
| **Отказоустойчивость мастеров** | Нет (1 мастер) | Есть (3 мастера) | Надежность↑ |

## ⚠️ Важные замечания

1. **Запуск займет больше времени** — инициализация 3 мастеров + большие кэши
2. **Нужно дождаться полной инициализации** — минимум 3-4 минуты
3. **Проверьте свободную память** после запуска:
   ```bash
   free -h
   docker stats --no-stream
   ```

## 🎯 Рекомендация: поэтапное внедрение

1. **Сначала** — просто увеличьте память в текущем конфиге (пункт 2)
2. **Затем** — добавьте 3 мастер-узла (пункт 1)  
3. **Позже** — оптимизируйте кэши и шарды (пункты 3-4)
