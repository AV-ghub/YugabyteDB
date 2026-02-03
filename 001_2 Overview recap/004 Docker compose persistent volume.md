```
да, надо попробовать вариант, который позволит:
"Данные будут сохраняться между перезапусками docker-compose down и docker-compose up"
так понимаю тот конфиг, который сейчас, не позволит это сделать?
т.е. вот та пользовательская табличка, которую мы создавали, она после docker-compose down исчезнет?
если да, то нужно это поправить с минимальными издержками
т.е. сделать под полным контролем докера, но чтобы данные сохранялись для последующих экспериментов с кластером
```

**Да, верно!** В текущем конфиге данные **исчезнут** после `docker-compose down`. Это "эпиhemeral storage" — временное хранение только на время жизни контейнера.

### ✅ Минимальное исправление: добавляем тома Docker

Вот обновленный `docker-compose.yml`, который **минимально меняет текущую работу**, но добавляет сохранение данных. Я добавил всего **9 строк** в конце файла:

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
      - "--replication_factor=3"
    mem_limit: 1024m
    ports:
      - "7000:7000"
    volumes:
      - yb_master_data:/mnt/master   # <-- Добавил том для мастера

  yb-tserver-a:
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-tserver"
      - "--fs_data_dirs=/mnt/tserver"
      - "--tserver_master_addrs=yb-master:7100"
      - "--rpc_bind_addresses=yb-tserver-a:9100"
      - "--enable_ysql"
      - "--ysql_enable_auth=false"
      - "--pgsql_proxy_bind_address=0.0.0.0:5433"
    ports:
      - "5433:5433"
    depends_on:
      - yb-master
    mem_limit: 2048m
    volumes:
      - yb_tserver_a_data:/mnt/tserver   # <-- Том для TServer A

  yb-tserver-b:
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-tserver"
      - "--fs_data_dirs=/mnt/tserver"
      - "--tserver_master_addrs=yb-master:7100"
      - "--rpc_bind_addresses=yb-tserver-b:9100"
      - "--enable_ysql"
      - "--ysql_enable_auth=false"
      - "--pgsql_proxy_bind_address=0.0.0.0:5433"
    ports:
      - "5434:5433"
    depends_on:
      - yb-master
    mem_limit: 2048m
    volumes:
      - yb_tserver_b_data:/mnt/tserver   # <-- Том для TServer B

  yb-tserver-c:
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-tserver"
      - "--fs_data_dirs=/mnt/tserver"
      - "--tserver_master_addrs=yb-master:7100"
      - "--rpc_bind_addresses=yb-tserver-c:9100"
      - "--enable_ysql"
      - "--ysql_enable_auth=false"
      - "--pgsql_proxy_bind_address=0.0.0.0:5433"
    ports:
      - "5435:5433"
    depends_on:
      - yb-master
    mem_limit: 2048m
    volumes:
      - yb_tserver_c_data:/mnt/tserver   # <-- Том для TServer C

# === ВСЕГО 9 НОВЫХ СТРОК ВНИЗУ ===
volumes:
  yb_master_data:
  yb_tserver_a_data:
  yb_tserver_b_data:
  yb_tserver_c_data:
```

### 🚀 Что это даёт:

1. **Данные сохраняются** между `docker-compose down` и `docker-compose up`
2. **Полный контроль Docker** — тома управляются самой системой
3. **Минимальные изменения** — добавляем только секцию `volumes:` в описания сервисов и блок `volumes:` в конце

### 🔧 Как перейти на новую схему:

```bash
# 1. Перейдите в директорию с вашим проектом
cd ~/yugabyte-prod

# 2. Остановите текущий кластер (БЕЗ -v!)
docker-compose down

# 3. Обновите docker-compose.yml (скопируйте конфиг выше)

# 4. Запустите снова
docker-compose up -d
sleep 120

# 5. Проверьте, что данные сохранились
psql -h localhost -p 5433 -U yugabyte -c "SELECT * FROM cluster_test;"
```

### 📊 Как это работает под капотом:

| Компонент | Что хранит | Где физически (на CentOS 7) |
|-----------|------------|----------------------------|
| `yb_master_data` | Метаданные кластера | `/var/lib/docker/volumes/yugabyte-prod_yb_master_data/_data` |
| `yb_tserver_a_data` | Пользовательские данные TServer A | `/var/lib/docker/volumes/yugabyte-prod_yb_tserver_a_data/_data` |
| `yb_tserver_b_data` | Пользовательские данные TServer B | `/var/lib/docker/volumes/yugabyte-prod_yb_tserver_b_data/_data` |
| `yb_tserver_c_data` | Пользовательские данные TServer C | `/var/lib/docker/volumes/yugabyte-prod_yb_tserver_c_data/_data` |

### 🔍 Проверить, что тома созданы:

```bash
# Посмотреть все тома проекта
docker volume ls | grep yugabyte-prod

# Увидеть физический путь
docker volume inspect yugabyte-prod_yb_tserver_a_data | grep Mountpoint
```

### ⚠️ Важные нюансы:

1. **Чтобы УДАЛИТЬ данные** — используйте `docker-compose down -v` (с флагом `-v`)
2. **Обновление конфига** — если вы меняете параметры, тома сохранят старые данные
3. **Миграция на другой диск** — можно позже, когда понадобится

### ✅ Практическая проверка:

Создайте тестовый сценарий:
```bash
# 1. Создайте таблицу
psql -h localhost -p 5433 -U yugabyte -c "CREATE TABLE persistent_test (id SERIAL PRIMARY KEY, value TEXT);"
psql -h localhost -p 5433 -U yugabyte -c "INSERT INTO persistent_test (value) VALUES ('До остановки');"

# 2. Остановите кластер
docker-compose down

# 3. Запустите снова
docker-compose up -d
sleep 120

# 4. Проверьте
psql -h localhost -p 5433 -U yugabyte -c "SELECT * FROM persistent_test;"
```

**Данные должны сохраниться!** Теперь вы можете останавливать и запускать кластер сколько угодно, и ваши таблицы останутся.
