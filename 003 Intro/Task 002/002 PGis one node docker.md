```
репозитории проверил, все хорошо
docker image inspect postgis/postgis:latest | grep -A5 -B5 PG_VERSION
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/lib/postgresql/17/bin",
                "GOSU_VERSION=1.17",
                "LANG=en_US.utf8",
                "PG_MAJOR=17",
                "PG_VERSION=17.5-1.pgdg110+1",
                "PGDATA=/var/lib/postgresql/data",
                "POSTGIS_MAJOR=3",
                "POSTGIS_VERSION=3.5.2+dfsg-1.pgdg110+1"

есть пара предложений
можешь код, который в "Пример Dockerfile:" дать сразу через создание файлика?
т.е. через << EOF
...
EOF

и второе
а мы можем контейнер собрать так, что бы запустить его не композом, а в одноузловом варианте, просто докером?
чтобы не курочить рабочий кластер и быстрее теститься и править ошибки?
```

Отличный подход! Работать с одноузловым контейнером для тестирования — это идеальный способ быстро итерироваться. Давайте реализуем всё по шагам.

## 🐳 Этап 3.1: Создание Dockerfile для одноузлового теста

Сначала создадим рабочий файл Dockerfile прямо через терминал:

```bash
# 1. Создаем директорию для сборки
mkdir -p ~/yugabyte-postgis-test
cd ~/yugabyte-postgis-test

# 2. Создаем Dockerfile с мультистейдж-сборкой
cat > Dockerfile << 'EOF'
# Первый этап: Установка PostGIS в среде, совместимой с YugabyteDB
FROM yugabytedb/yugabyte:2.15.3.2-b1 as postgis-builder

RUN yum update -y && \
    yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm && \
    yum install -y postgresql11-server postgis31_11 postgis31_11-client

# Копируем файлы PostGIS в стандартные каталоги YugabyteDB
RUN cp -r /usr/pgsql-11/lib/postgis* /home/yugabyte/postgres/lib/ && \
    cp -r /usr/pgsql-11/share/extension/postgis* /home/yugabyte/postgres/share/extension/

# Второй этап: Финальный образ
FROM yugabytedb/yugabyte:2.19.2.0-b121

# Копируем только нужные файлы из первого этапа
COPY --from=postgis-builder /home/yugabyte/postgres/lib/postgis* /home/yugabyte/postgres/lib/
COPY --from=postgis-builder /home/yugabyte/postgres/share/extension/postgis* /home/yugabyte/postgres/share/extension/

# Выполняем перелинковку библиотек
RUN /home/yugabyte/bin/post_install.sh -e

# Дополнительно: можно установить недостающие библиотеки, если понадобится
# RUN yum install -y libicu
EOF

# 3. Смотрим, что создалось
cat Dockerfile
```

**Важное замечание по версиям**: Я использовал `2.15.3.2-b1` как builder-образ, потому что он основан на CentOS 7, что гарантирует совместимость библиотек. Финальный образ — `2.19.2.0-b121` (последняя стабильная). Если хотите другую версию, проверьте в [официальном репозитории](https://hub.docker.com/r/yugabytedb/yugabyte/tags).

## 🚀 Этап 3.2: Сборка и запуск одноузлового контейнера

Теперь собираем образ и запускаем его в одноузловом режиме:

```bash
# 1. Собираем образ (это займет 10-15 минут)
docker build -t yugabyte-with-postgis-test:latest .

# 2. Запускаем контейнер в одноузловом режиме
#    Пробрасываем порты: 5433 (YSQL), 9042 (YCQL), 7000 (Web UI)
docker run -d \
  --name yb-single-node-test \
  -p 5433:5433 \
  -p 9042:9042 \
  -p 7000:7000 \
  -p 9000:9000 \
  yugabyte-with-postgis-test:latest \
  bin/yugabyted start \
  --base_dir=/home/yugabyte/yb_data \
  --daemon=false

# 3. Ждем 30-60 секунд, пока кластер запустится
sleep 60

# 4. Проверяем статус
docker exec yb-single-node-test bin/yugabyted status
```

## 🧪 Этап 3.3: Тестирование PostGIS в одноузловом контейнере

```bash
# 1. Подключаемся к YSQL и пробуем установить PostGIS
docker exec -it yb-single-node-test bash << 'EOF'
echo "=== Проверка версии PostgreSQL в YugabyteDB ==="
./bin/postgres --version

echo -e "\n=== Попытка создания расширения PostGIS ==="
./bin/ysqlsh -h localhost -p 5433 -U yugabyte -c "CREATE EXTENSION postgis;"

echo -e "\n=== Проверка установки ==="
./bin/ysqlsh -h localhost -p 5433 -U yugabyte -c "SELECT PostGIS_Full_Version();"

echo -e "\n=== Проверка доступных расширений ==="
./bin/ysqlsh -h localhost -p 5433 -U yugabyte -c "SELECT * FROM pg_extension WHERE extname LIKE 'postgis%';"
EOF
```

## 🔧 Этап 3.4: Быстрая диагностика при ошибках

Если возникнут ошибки при создании расширения, вот команды для диагностики прямо в контейнере:

```bash
# Заходим в контейнер для диагностики
docker exec -it yb-single-node-test bash

# Внутри контейнера:
# 1. Проверяем, скопировались ли файлы PostGIS
ls -la /home/yugabyte/postgres/lib/postgis* | head -5
ls -la /home/yugabyte/postgres/share/extension/postgis* | head -5

# 2. Проверяем зависимости библиотек
ldd /home/yugabyte/postgres/lib/postgis-3.so 2>/dev/null | head -20

# 3. Ищем ошибки в логах
tail -100 /home/yugabyte/yb_data/node-1/disk-1/yb-data/tserver/logs/postgresql-*.log | grep -i postgis

# 4. Проверяем, какие библиотеки уже есть
ls -la /home/yugabyte/postgres/lib/ | grep -E '(sqlite|gdal|proj|geos)'

# 5. Альтернатива: установка недостающих библиотек прямо в контейнере
# yum install -y epel-release
# yum install -y libsqlite3x-devel gdal38-libs proj proj-nad geos38
```

## 📝 Пример рабочего теста PostGIS

Если расширение создалось успешно, выполните тестовый геозапрос:

```bash
docker exec yb-single-node-test ./bin/ysqlsh -h localhost -p 5433 -U yugabyte << 'SQL'
-- Простой тест PostGIS
CREATE TABLE test_geometries (
    id SERIAL PRIMARY KEY,
    name TEXT,
    geom geometry(Point, 4326)
);

INSERT INTO test_geometries (name, geom) VALUES
    ('Точка 1', ST_SetSRID(ST_MakePoint(37.617, 55.755), 4326)),
    ('Точка 2', ST_SetSRID(ST_MakePoint(30.314, 59.939), 4326));

-- Проверяем расстояние между точками
SELECT 
    a.name as point_a,
    b.name as point_b,
    ST_Distance(a.geom::geography, b.geom::geography) / 1000 as distance_km
FROM test_geometries a
CROSS JOIN test_geometries b
WHERE a.id < b.id;

-- Удаляем тестовую таблицу
DROP TABLE test_geometries;
SQL
```

## 💡 Ключевые моменты для быстрой итерации

1. **Быстрая пересборка**: Если нужно внести изменения в Dockerfile:
   ```bash
   docker stop yb-single-node-test
   docker rm yb-single-node-test
   docker build -t yugabyte-with-postgis-test:latest .
   # Повторить команду docker run из Этапа 3.2
   ```

2. **Сохранить данные между запусками** (если нужно):
   ```bash
   # Добавить volume при запуске
   docker run -d \
     --name yb-single-node-test \
     -v yb_test_data:/home/yugabyte/yb_data \
     -p 5433:5433 \
     -p 9042:9042 \
     -p 7000:7000 \
     yugabyte-with-postgis-test:latest \
     bin/yugabyted start --daemon=false
   ```

3. **Проверка через Web UI**: Откройте в браузере `http://localhost:7000` чтобы увидеть статус кластера.

## 🚨 Возможные проблемы и их решения

| Проблема | Причина | Решение |
|----------|---------|---------|
| `ERROR: could not load library "/home/yugabyte/postgres/lib/postgis-3.so"` | Несовместимость GLIBC | Использовать CentOS 7 в builder-этапе (как в примере) |
| `libsqlite3.so.0: cannot open shared object file` | Не хватает библиотек | Установить в builder-этапе: `yum install -y sqlite-devel` |
| Расширение создается, но функции не работают | Не все файлы скопированы | Проверить копирование всех `postgis*` файлов |
| Контейнер не запускается | Конфликт портов | Использовать другие порты: `-p 5434:5433 -p 9043:9042` |

## ✅ Что дальше?

1. **Сначала запустите сборку** (`docker build`). Она займет время, но вы сможете заниматься другими делами.
2. **Затем протестируйте** одноузловый контейнер.
3. **Если всё работает** — можно адаптировать Dockerfile для docker-compose вашего рабочего кластера.
