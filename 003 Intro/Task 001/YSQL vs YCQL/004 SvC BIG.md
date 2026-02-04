```
ну вот так у меня получилось
docker exec -it yugabyte_yb-tserver1_1 bash -c "
 echo 'Creating CSV file...'
 echo 'bucket,item_id,value,payload' > /tmp/bulk_data.csv
 for i in \$(seq 1 100000); do
   bucket=\$((i % 1000))
   item_id=\"item_\$i\"
   # Используем целые числа для упрощения
   value=\$((RANDOM % 10000))  # 0-9999
   payload=\"payload_\$i\"
   echo \"\$bucket,\$item_id,\$value,\$payload\" >> /tmp/bulk_data.csv
 done
 echo 'CSV created with 100K rows'
 echo 'First 3 lines:'
 head -3 /tmp/bulk_data.csv
 "

Creating CSV file...
CSV created with 100K rows
First 3 lines:
bucket,item_id,value,payload
1,item_1,8717,payload_1
2,item_2,5098,payload_2



# Загружаем через ycqlsh
docker exec -it yugabyte_yb-tserver1_1 /home/yugabyte/bin/ycqlsh -e "
 USE stress;
 COPY ycql_perf_test (bucket, item_id, value, payload)
 FROM '/tmp/bulk_data.csv'
 WITH HEADER = true
 AND MAXATTEMPTS = 5
 AND CHUNKSIZE = 1000
 AND INGESTRATE = 1000;
 "
Connection error: ('Unable to connect to any servers', {'127.0.0.1:9042': ConnectionRefusedError(111, "Tried connecting to [('127.0.0.1', 9042)]. Last error: Connection refused")})


docker exec -it yugabyte_yb-tserver1_1 /home/yugabyte/bin/ycqlsh 172.26.0.4 -e "
 USE stress;
 COPY ycql_perf_test (bucket, item_id, value, payload)
 FROM '/tmp/bulk_data.csv'
 WITH HEADER = true
 AND MAXATTEMPTS = 5
 AND CHUNKSIZE = 1000
 AND INGESTRATE = 1000;
 "

Using 11 child processes

Starting copy of stress.ycql_perf_test with columns [bucket, item_id, value, payload].
Processed: 100000 rows; Rate:     497 rows/s; Avg. rate:     999 rows/s
100000 rows imported from 1 files in 0 day, 0 hour, 1 minutes, and 40.106 seconds (0 skipped).


# 1. Есть ли таблица?
docker exec -it yugabyte_yb-tserver1_1 /home/yugabyte/bin/ycqlsh 172.26.0.4 -e "DESC TABLE stress.ycql_perf_test;"

CREATE TABLE stress.ycql_perf_test (
    bucket int,
    item_id text,
    value double,
    payload text,
    PRIMARY KEY ((bucket, item_id))
) WITH default_time_to_live = 0
    AND transactions = {'enabled': 'false'};


# 2. Доступен ли порт 9042?
[root@mongo-db3 yugabyte]# docker exec -it yugabyte_yb-tserver1_1 netstat -tlnp | grep 9042
tcp        0      0 172.26.0.4:9042         0.0.0.0:*               LISTEN      7/yb-tserver


docker exec -it yugabyte_yb-tserver1_1 bash
[root@9b1281140ed8 yugabyte]#

[root@9b1281140ed8 yugabyte]# netstat -tlnp | grep 9042
tcp        0      0 172.26.0.4:9042         0.0.0.0:*               LISTEN      7/yb-tserver

[root@9b1281140ed8 yugabyte]# hostname -I
172.26.0.4

[root@9b1281140ed8 yugabyte]# /home/yugabyte/bin/ycqlsh 172.26.0.4
Connected to local cluster at 172.26.0.4:9042.
[ycqlsh 5.0.1 | Cassandra 3.9-SNAPSHOT | CQL spec 3.4.2 | Native protocol v4]
Use HELP for help.
ycqlsh>

ycqlsh> CREATE KEYSPACE IF NOT EXISTS stress WITH REPLICATION = {'class': 'SimpleStrategy', 'replication_factor': 3};
ycqlsh> USE stress;


ycqlsh:stress> DESC TABLE stress.ycql_perf_test;

CREATE TABLE stress.ycql_perf_test (
    bucket int,
    item_id text,
    value double,
    payload text,
    PRIMARY KEY ((bucket, item_id))
) WITH default_time_to_live = 0
    AND transactions = {'enabled': 'false'};

ycqlsh:stress> select count(*) from ycql_perf_test;

 count
--------
 100002

(1 rows)

а что теперь можно показательно протестировать на этих данных?
так, чтобы показать преимущества и сильные стороны YCQL?
что можешь предложить в качестве тестов?
```


Отлично! Вы успешно загрузили 100K строк. Теперь покажем **сильные стороны YCQL** на этих данных. Вот набор показательных тестов, которые демонстрируют, для чего оптимизирован YCQL.

## 🎯 Тест 1: Молниеносные точечные запросы по ключу (главная сила YCQL)

```sql
-- Включим трассировку для измерения
\T on

-- 1. Запрос по ПОЛНОМУ первичному ключу (самая быстрая операция)
SELECT * FROM ycql_perf_test WHERE bucket = 123 AND item_id = 'item_456';

-- 2. Запрос только по ЧАСТИ ключа (bucket - часть ключа партиции)
SELECT * FROM ycql_perf_test WHERE bucket = 123 LIMIT 10;

-- 3. Сравним: запрос по НЕключевому полю (если бы был индекс)
-- Сначала создадим индекс (может не работать для нетранзакционной таблицы)
CREATE INDEX IF NOT EXISTS ON ycql_perf_test (value);

-- Затем выполним (будет медленно)
SELECT * FROM ycql_perf_test WHERE value = 5000 LIMIT 5;

\T off
```

**Что смотреть в выводе TRACING:**
- Время выполнения (`elapsed`) для запроса 1 будет **< 1ms**
- Запрос 2 также быстрый, так как фильтр по ключу партиции
- Запрос 3 (если индекс создастся) будет значительно медленнее

## 📊 Тест 2: Массовые операции в пределах одной партиции

```sql
-- Подготовим данные в одной партиции для демонстрации
INSERT INTO ycql_perf_test (bucket, item_id, value, payload) 
VALUES (999, 'batch_item_1', 100.0, 'test_1');
INSERT INTO ycql_perf_test (bucket, item_id, value, payload) 
VALUES (999, 'batch_item_2', 200.0, 'test_2');
INSERT INTO ycql_perf_test (bucket, item_id, value, payload) 
VALUES (999, 'batch_item_3', 300.0, 'test_3');

-- 1. Чтение всех записей одной партиции (очень быстро)
SELECT COUNT(*) FROM ycql_perf_test WHERE bucket = 999;
SELECT * FROM ycql_perf_test WHERE bucket = 999;

-- 2. Множественное обновление в одной партиции
UPDATE ycql_perf_test SET value = value + 1 WHERE bucket = 999 AND item_id = 'batch_item_1';
UPDATE ycql_perf_test SET value = value + 1 WHERE bucket = 999 AND item_id = 'batch_item_2';
```

## ⚡ Тест 3: Параллельные чтения (демонстрация горизонтального масштабирования)

```sql
-- Чтение из разных партиций параллельно (имитируем реальную нагрузку)
-- В реальности это делается из приложения, но в ycqlsh можно показать концепцию

-- 1. Быстрое чтение из случайных партиций
SELECT * FROM ycql_perf_test WHERE bucket = 1 AND item_id = 'item_50';
SELECT * FROM ycql_perf_test WHERE bucket = 500 AND item_id = 'item_25000';
SELECT * FROM ycql_perf_test WHERE bucket = 750 AND item_id = 'item_75000';

-- 2. Проверка распределения данных
-- Посмотрим, как данные распределены по партициям
SELECT bucket, COUNT(*) as record_count 
FROM ycql_perf_test 
WHERE bucket IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10) 
GROUP BY bucket;
```

## 🔄 Тест 4: Сравнение с "плохими" запросами (что НЕ стоит делать в YCQL)

```sql
\T on

-- 1. ПОЛНОЕ сканирование таблицы (антипаттерн для YCQL)
SELECT COUNT(*) FROM ycql_perf_test;

-- 2. Агрегация без фильтра по ключу партиции
SELECT bucket, COUNT(*) FROM ycql_perf_test 
WHERE bucket IN (1, 2, 3) 
GROUP BY bucket;

-- 3. Попытка SELECT DISTINCT (не поддерживается в YCQL)
SELECT DISTINCT bucket FROM ycql_perf_test LIMIT 10;

\T off
```

## 📈 Тест 5: Работа с TTL (Time-To-Live) - уникальная возможность YCQL

```sql
-- 1. Создадим таблицу с TTL
CREATE TABLE test_ttl (
    id text PRIMARY KEY,
    data text,
    created_at timestamp
) WITH default_time_to_live = 10;  -- Автоудаление через 10 секунд!

-- 2. Вставим данные
INSERT INTO test_ttl (id, data, created_at) 
VALUES ('temp1', 'Эта запись исчезнет через 10 сек', toTimestamp(now()));

-- 3. Проверим сразу
SELECT * FROM test_ttl;

-- 4. Подождем 15 секунд и проверим снова
SELECT * FROM test_ttl;  -- Данных не будет!

-- 5. TTL на уровне отдельной записи
INSERT INTO test_ttl (id, data, created_at) 
VALUES ('temp2', 'Эта исчезнет через 30 сек', toTimestamp(now())) 
USING TTL 30;
```

## 🎲 Тест 6: Использование встроенных функций и batch-операций

```sql
-- 1. Использование функций CQL
SELECT 
    toDate(toTimestamp(now())) as today,
    blobAsText(textAsBlob('Hello YCQL')) as converted,
    uuid() as new_uuid
FROM system.local;

-- 2. Обновление с условием (если поддерживается)
UPDATE ycql_perf_test 
SET value = value * 1.1 
WHERE bucket = 100 AND item_id = 'item_500' 
IF value > 50;

-- 3. Чтение с пагинацией (очень эффективно в YCQL)
SELECT * FROM ycql_perf_test WHERE bucket = 100 LIMIT 10;

-- Сохраним paging state для следующей "страницы"
-- (в реальном приложении это делается через драйвер)
```

## 📋 Тест 7: Проверка метрик и производительности

```sql
-- 1. Проверим размер данных
SELECT 
    COUNT(*) as total_rows,
    MIN(value) as min_value,
    MAX(value) as max_value,
    AVG(value) as avg_value
FROM ycql_perf_test 
WHERE bucket = 1;

-- 2. Профилируем разницу в скорости
\T on
-- Быстрый запрос
SELECT * FROM ycql_perf_test WHERE bucket = 1 AND item_id = 'item_1';
-- Медленный запрос (сканирование)
SELECT * FROM ycql_perf_test WHERE value = 5000 LIMIT 1;
\T off
```

## 🏆 Финальный демонстрационный тест: Имитация реального сценария

```sql
-- Создадим модель пользовательских сессий (классический use case для YCQL)
CREATE TABLE user_sessions (
    user_id text,
    session_id timeuuid,
    page_views list<text>,
    session_data map<text, text>,
    expires_at timestamp,
    PRIMARY KEY ((user_id), session_id)
) WITH CLUSTERING ORDER BY (session_id DESC)
  AND default_time_to_live = 86400;  -- Автоудаление через 24 часа

-- 1. Вставка сессии
INSERT INTO user_sessions (user_id, session_id, page_views, session_data, expires_at)
VALUES (
    'user_123', 
    now(), 
    ['/home', '/products', '/cart'],
    {'browser': 'Chrome', 'ip': '192.168.1.1', 'status': 'active'},
    toTimestamp(now() + 3600000)  -- +1 час
);

-- 2. Быстрое получение последних сессий пользователя
SELECT * FROM user_sessions WHERE user_id = 'user_123' LIMIT 5;

-- 3. Обновление сессии
UPDATE user_sessions 
SET page_views = page_views + ['/checkout'],
    session_data = session_data + {'status': 'checkout'}
WHERE user_id = 'user_123' AND session_id = [ВСТАВЬТЕ_РЕАЛЬНЫЙ_UUID];

-- 4. Поиск по данным внутри map (если нужно)
SELECT * FROM user_sessions 
WHERE user_id = 'user_123' 
  AND session_data['status'] = 'active' 
ALLOW FILTERING;  -- Внимание: ALLOW FILTERING медленный!
```

## 📊 Что показывают эти тесты:

| Тест | Демонстрируемая сила YCQL | Ожидаемое время | Почему это важно |
|------|--------------------------|-----------------|------------------|
| **Тест 1** | Молниеносные точечные запросы | < 1ms | Основа производительности |
| **Тест 2** | Эффективная работа в пределах партиции | < 5ms | Паттерн для агрегации |
| **Тест 5** | TTL - автоуправление данными | - | Уникальная NoSQL-фича |
| **Тест 6** | Встроенные функции, Batch | < 10ms | Богатый функционал |
| **User Sessions** | Гибкая схема (list, map) | < 2ms | Реальный use case |

## 🎯 Рекомендация для быстрой демонстрации:

Выполните эти 3 ключевых теста за 10 минут:

1. **Тест 1** с `\T on` - покажет скорость точечных запросов
2. **Тест 5** с TTL - покажет уникальную NoSQL-возможность  
3. **User Sessions пример** - покажет реальный use case

**Главный вывод для демонстрации:** YCQL блестяще справляется с **точечными операциями по ключу** и **работе с гибкой схемой**, но не предназначен для аналитических запросов с полным сканированием таблицы.

Какой тест хотите попробовать первым?
