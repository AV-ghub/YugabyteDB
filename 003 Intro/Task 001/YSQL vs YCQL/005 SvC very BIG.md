```
а можем мы нагенерить теперь тех же данных, но 100 млн строк?
так понимаю если делать это через скрипт
docker exec -it yugabyte_yb-tserver1_1 bash -c "
 echo 'Creating CSV file...'
 echo 'bucket,item_id,value,payload' > /tmp/bulk_data.csv
 for i in \$(seq 1 100000000); do
   bucket=\$((i % 1000))
   item_id=\"item_\$i\"
   # Используем целые числа для упрощения
   value=\$((RANDOM % 10000))  # 0-9999
   payload=\"payload_\$i\"
   echo \"\$bucket,\$item_id,\$value,\$payload\" >> /tmp/bulk_data.csv
 done
 echo 'CSV created with 100K rows'
 echo 'First 3 lines:'
 head -3 /tmp/big_data.csv
 "
то csv формат такого не выдержит
или это для него будет нормально?
данные мы потом сможем загрузить через
docker exec -it yugabyte_yb-tserver1_1 /home/yugabyte/bin/ycqlsh 172.26.0.4 -e "
 USE stress;
 COPY ycql_perf_test (bucket, item_id, value, payload)
 FROM '/tmp/bulk_data.csv'
 WITH HEADER = true
 AND MAXATTEMPTS = 5
 AND CHUNKSIZE = 1000
 AND INGESTRATE = 1000;
 "
?
это реально?
```

**Технически возможно, но практические ограничения делают это нецелесообразным.**

Давайте разберем почему и как можно обойти проблему.

## 🚫 Почему ваш подход НЕ сработает для 100M строк:

### 1. **Размер файла будет огромным**
- 100M строк × ~50 байт/строка = **~5 ГБ** CSV файл
- Docker контейнеры обычно имеют ограниченное дисковое пространство в `/tmp`
- Генерация через bash цикл займет **часы, если не дни**

### 2. **Проблемы с bash-генерацией:**
- `seq 1 100000000` создаст список из 100M чисел в памяти
- `RANDOM % 10000` ограничено 32767, плохое распределение
- Эхо в файл для каждой строки - невероятно медленно

### 3. **Проблемы с загрузкой:**
- `COPY` в ycqlsh может упасть на таймаутах
- Одна транзакция на весь файл - риск потери данных
- Нет контроля над процессом

## ✅ Практические способы загрузки 100M строк

### **Способ 1: Постепенная загрузка частями (рекомендуется)**

```bash
# Генерируем и загружаем по 1M строк за раз
for batch in {1..100}; do
    echo "Processing batch $batch/100..."
    
    # Генерация 1M строк через Python (быстро)
    docker exec yugabyte_yb-tserver1_1 python3 << EOF
import random, csv, time
batch_num = $batch
start = (batch_num - 1) * 1000000 + 1
end = batch_num * 1000000

with open(f'/tmp/batch_{batch_num:03d}.csv', 'w') as f:
    writer = csv.writer(f)
    writer.writerow(['bucket', 'item_id', 'value', 'payload'])
    
    chunk_size = 100000
    for chunk_start in range(start, end + 1, chunk_size):
        chunk_end = min(chunk_start + chunk_size - 1, end)
        rows = []
        for i in range(chunk_start, chunk_end + 1):
            rows.append([i % 1000, f'item_{i}', random.randint(0, 9999), f'payload_{i}'])
        writer.writerows(rows)
        
        if chunk_start % 1000000 == 0:
            print(f"Generated {chunk_start:,} rows...")

print(f"Batch {batch_num}: Generated {end-start+1:,} rows")
EOF

    # Загрузка батча
    docker exec yugabyte_yb-tserver1_1 /home/yugabyte/bin/ycqlsh 172.26.0.4 -e "
USE stress;
COPY ycql_perf_test (bucket, item_id, value, payload) 
FROM '/tmp/batch_${batch:03d}.csv' 
WITH HEADER = true 
AND CHUNKSIZE = 1000 
AND INGESTRATE = 5000;
"
    
    # Очистка файла
    docker exec yugabyte_yb-tserver1_1 rm -f /tmp/batch_${batch:03d}.csv
    
    echo "Batch $batch loaded"
done
```

### **Способ 2: Использование специализированного инструмента**

```bash
# 1. Установите yb-bulk-load (специально для YugabyteDB)
docker exec yugabyte_yb-tserver1_1 bash -c "
wget -O /tmp/yb-bulk-load https://github.com/yugabyte/yb-bulk-load/releases/download/v1.0.0/yb-bulk-load
chmod +x /tmp/yb-bulk-load
"

# 2. Создайте конфиг
cat > bulk_load_config.json << EOF
{
  "tableName": "stress.ycql_perf_test",
  "schema": "bucket INT, item_id TEXT, value DOUBLE, payload TEXT",
  "columnMapping": ["bucket", "item_id", "value", "payload"],
  "fileFormat": "CSV",
  "csvFormat": {
    "delimiter": ",",
    "header": true
  },
  "connection": {
    "hosts": ["172.26.0.4:9042"],
    "keyspace": "stress"
  },
  "batchSize": 1000,
  "parallelism": 10
}
EOF

# 3. Запустите генератор + загрузку параллельно
docker exec yugabyte_yb-tserver1_1 python3 << 'EOF'
import subprocess, random, tempfile, os
from concurrent.futures import ThreadPoolExecutor

def generate_and_load(start, end, batch_id):
    # Генерация в памяти и прямая отправка
    import io, csv
    from cassandra.cluster import Cluster
    
    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(['bucket', 'item_id', 'value', 'payload'])
    
    for i in range(start, end + 1):
        writer.writerow([i % 1000, f'item_{i}', random.random() * 10000, f'payload_{i}'])
        if (i - start + 1) % 100000 == 0:
            print(f"Batch {batch_id}: Generated {i-start+1:,} rows")
    
    output.seek(0)
    
    # Прямая вставка через драйвер
    cluster = Cluster(['172.26.0.4'], port=9042)
    session = cluster.connect('stress')
    
    # Используем batch-вставки
    from cassandra.query import BatchStatement
    insert_stmt = session.prepare(
        "INSERT INTO ycql_perf_test (bucket, item_id, value, payload) VALUES (?, ?, ?, ?)"
    )
    
    reader = csv.reader(output)
    next(reader)  # Skip header
    
    batch_size = 100
    batch = BatchStatement()
    count = 0
    
    for row in reader:
        batch.add(insert_stmt, (int(row[0]), row[1], float(row[2]), row[3]))
        count += 1
        
        if count % batch_size == 0:
            session.execute(batch)
            batch = BatchStatement()
    
    if count % batch_size != 0:
        session.execute(batch)
    
    print(f"Batch {batch_id}: Loaded {count:,} rows")
    return count

# Параллельная обработка 100M строк
total_rows = 100_000_000
batch_size = 1_000_000
num_batches = total_rows // batch_size

with ThreadPoolExecutor(max_workers=5) as executor:
    futures = []
    for batch in range(num_batches):
        start = batch * batch_size + 1
        end = (batch + 1) * batch_size
        futures.append(executor.submit(generate_and_load, start, end, batch))
    
    total_loaded = sum(f.result() for f in futures)
    print(f"Total loaded: {total_loaded:,} rows")
EOF
```

### **Способ 3: Использование Apache Spark (для production)**

```python
# spark_bulk_load.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, lit, rand, expr
import random

spark = SparkSession.builder \
    .appName("YCQL Bulk Load") \
    .config("spark.cassandra.connection.host", "172.26.0.4") \
    .config("spark.cassandra.connection.port", "9042") \
    .getOrCreate()

# Генерация 100M строк в Spark
df = spark.range(1, 100000001) \
    .select(
        (col("id") % 1000).alias("bucket"),
        expr("concat('item_', id)").alias("item_id"),
        (rand() * 10000).cast("double").alias("value"),
        expr("concat('payload_', id)").alias("payload")
    )

# Запись в YCQL
df.write \
    .format("org.apache.spark.sql.cassandra") \
    .mode("append") \
    .options(table="ycql_perf_test", keyspace="stress") \
    .save()

print("100M rows loaded!")
```

## ⚡ Реалистичный тест за разумное время

**Для демонстрации рекомендую начать с 10M строк:**

```bash
# Быстрая генерация и загрузка 10M строк
docker exec yugabyte_yb-tserver1_1 python3 << 'EOF'
from cassandra.cluster import Cluster
from cassandra.query import BatchStatement
import random, time

cluster = Cluster(['172.26.0.4'], port=9042)
session = cluster.connect('stress')

insert_stmt = session.prepare(
    "INSERT INTO ycql_perf_test (bucket, item_id, value, payload) VALUES (?, ?, ?, ?)"
)

start_time = time.time()
total_rows = 10_000_000
batch_size = 500

print(f"Starting load of {total_rows:,} rows...")

for i in range(1, total_rows + 1):
    bucket = i % 1000
    item_id = f'item_{i}'
    value = random.random() * 10000
    payload = f'payload_{i}'
    
    if i % batch_size == 1:
        batch = BatchStatement()
    
    batch.add(insert_stmt, (bucket, item_id, value, payload))
    
    if i % batch_size == 0:
        session.execute(batch)
    
    if i % 100000 == 0:
        elapsed = time.time() - start_time
        rate = i / elapsed
        print(f"Progress: {i:,} rows, Rate: {rate:,.0f} rows/sec, Elapsed: {elapsed:.1f}s")

elapsed = time.time() - start_time
print(f"\nCompleted: {total_rows:,} rows in {elapsed:.1f} seconds")
print(f"Average rate: {total_rows/elapsed:,.0f} rows/sec")
EOF
```

## 📊 Ожидаемая производительность:

| Объем | Метод | Время | Скорость | Рекомендация |
|-------|-------|-------|----------|--------------|
| **100K** | COPY из CSV | 1-2 мин | ~1K rows/sec | Для тестирования |
| **10M** | Python + Batch | 10-20 мин | 8-15K rows/sec | Для демо нагрузок |
| **100M** | Параллельный Python | 1.5-3 часа | 10-20K rows/sec | Максимальный тест |
| **100M+** | Apache Spark | 30-60 мин | 30-50K rows/sec | Production |

## 🎯 Практический совет:

**Начните с 10M строк** - это достаточно для демонстрации:
1. Покажет реальную производительность YCQL
2. Займет разумное время (15-30 минут)
3. Позволит провести все тесты производительности

```bash
# Быстрый старт: загрузите 10M строк
docker exec yugabyte_yb-tserver1_1 python3 << 'EOF'
# ... код для 10M строк выше
EOF

# Затем проведите тесты
docker exec -it yugabyte_yb-tserver1_1 /home/yugabyte/bin/ycqlsh 172.26.0.4 -e "
USE stress;
SELECT COUNT(*) FROM ycql_perf_test;
SELECT * FROM ycql_perf_test WHERE bucket = 1 AND item_id = 'item_50000';
"
```

**Вывод:** 100M строк загрузить можно, но лучше начать с 10M для демонстрации возможностей. Главное - не один огромный файл, а потоковая загрузка батчами.
