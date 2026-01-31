```bash
нет. к сожалению все пока вот так
andrey@andrey-VirtualBox:~/dcompose/yugabyte$ psql -h localhost -p 5433 -U yugabyte
psql (16.0 (Ubuntu 16.0-1.pgdg20.04+1), server 15.12-YB-2.25.2.0-b0)
Type "help" for help.

yugabyte=# SELECT * FROM system.local;
ERROR:  relation "system.local" does not exist
LINE 1: SELECT * FROM system.local;
                      ^
yugabyte=# \q
andrey@andrey-VirtualBox:~/dcompose/yugabyte$ docker compose restart yb-tserver-n1
WARN[0000] /home/andrey/dcompose/yugabyte/docker-compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Restarting 1/1
 ✔ Container yugabyte-yb-tserver-n1-1  Started                                                                                                                                                        0.7s 
andrey@andrey-VirtualBox:~/dcompose/yugabyte$ psql -h localhost -p 5433 -U yugabyte
psql (16.0 (Ubuntu 16.0-1.pgdg20.04+1), server 15.12-YB-2.25.2.0-b0)
Type "help" for help.

yugabyte=# SELECT * FROM system.local;
ERROR:  relation "system.local" does not exist
LINE 1: SELECT * FROM system.local;

при этом

$ docker exec yugabyte-yb-master-n1-1 yb-admin --master_addresses yb-master-n1:7100,yb-master-n2:7100,yb-master-n3:7100 list_all_masters
Master UUID                      	RPC Host/Port        	State    	Role 	Broadcast Host/Port 
a493005285e54eb598abbec610665d41 	yb-master-n1:7100    	ALIVE    	FOLLOWER 	yb-master-n1:7100   
47c54bb298494e479bef578658ad29f0 	yb-master-n2:7100    	ALIVE    	LEADER 	yb-master-n2:7100   
1a0bb08328d14a2f81263d081eba6ede 	yb-master-n3:7100    	ALIVE    	FOLLOWER 	yb-master-n3:7100
```

Отлично, что проверили! Проблема ясна: **мастер-узлы работают (есть лидер и фолловеры), но системный каталог YSQL не инициализирован**. Это происходит потому, что в вашем docker-compose файле, скорее всего, не включен YSQL или он не дождался инициализации.

### 🔧 Быстрое решение

Нужно **явно включить YSQL и пересоздать кластер** с правильными параметрами. Остановите и удалите текущий кластер:

```bash
# Останавливаем и удаляем все контейнеры и тома
docker-compose down -v
```

### 📋 Обновленный docker-compose.yml

Создайте новый файл `docker-compose.yml` с **явным включением YSQL** и **ожиданием готовности мастеров**:

```yaml
version: '3.8'

services:
  yb-master-n1:
    image: yugabytedb/yugabyte:latest
    command: [
      "/home/yugabyte/bin/yb-master",
      "--fs_data_dirs=/mnt/master",
      "--master_addresses=yb-master-n1:7100,yb-master-n2:7100,yb-master-n3:7100",
      "--rpc_bind_addresses=yb-master-n1:7100"
    ]
    ports:
      - "7000:7000"

  yb-master-n2:
    image: yugabytedb/yugabyte:latest
    command: [
      "/home/yugabyte/bin/yb-master",
      "--fs_data_dirs=/mnt/master",
      "--master_addresses=yb-master-n1:7100,yb-master-n2:7100,yb-master-n3:7100",
      "--rpc_bind_addresses=yb-master-n2:7100"
    ]

  yb-master-n3:
    image: yugabytedb/yugabyte:latest
    command: [
      "/home/yugabyte/bin/yb-master",
      "--fs_data_dirs=/mnt/master",
      "--master_addresses=yb-master-n1:7100,yb-master-n2:7100,yb-master-n3:7100",
      "--rpc_bind_addresses=yb-master-n3:7100"
    ]

  yb-tserver-n1:
    image: yugabytedb/yugabyte:latest
    command: [
      "/home/yugabyte/bin/yb-tserver",
      "--fs_data_dirs=/mnt/tserver",
      "--tserver_master_addrs=yb-master-n1:7100,yb-master-n2:7100,yb-master-n3:7100",
      "--rpc_bind_addresses=yb-tserver-n1:9100",
      "--enable_ysql=true",                     # Ключевой параметр!
      "--ysql_enable_auth=false",
      "--pgsql_proxy_bind_address=yb-tserver-n1:5433"
    ]
    ports:
      - "5433:5433"
    depends_on:
      - yb-master-n1
      - yb-master-n2
      - yb-master-n3
    # Ждем готовности мастеров перед запуском TServer
    healthcheck:
      test: ["CMD", "/home/yugabyte/bin/yb-admin", "--master_addresses", "yb-master-n1:7100,yb-master-n2:7100,yb-master-n3:7100", "list_all_masters"]
      interval: 10s
      timeout: 5s
      retries: 10
```

### 🚀 Запуск и проверка

```bash
# 1. Запускаем кластер
docker-compose up -d

# 2. Ждем 60-90 секунд для полной инициализации
sleep 90

# 3. Проверяем, что YSQL доступен
psql -h localhost -p 5433 -U yugabyte -c "SELECT version();"

# 4. Создаем тестовую таблицу
psql -h localhost -p 5433 -U yugabyte -c "CREATE TABLE users (id SERIAL PRIMARY KEY, name TEXT);"

# 5. Проверяем в веб-интерфейсе: http://localhost:7000
```

### 🔍 Если проблема останется: ручная инициализация

Если после этого YSQL все еще не работает, выполните ручную инициализацию:

```bash
# Войдите в контейнер TServer
docker exec -it yugabyte-yb-tserver-n1-1 bash

# Инициализируйте YSQL вручную (внутри контейнера)
/home/yugabyte/bin/initdb -D /mnt/tserver/ysql_data

# Перезапустите TServer
exit
docker-compose restart yb-tserver-n1
```

### 📊 Почему это важно

Параметр `--enable_ysql=true` критически важен — без него TServer не создает PostgreSQL-совместимый слой. В вашем предыдущем конфиге этого параметра, скорее всего, не было.

**Попробуйте с новым docker-compose.yml. Сообщите о результате!** Особенно важно увидеть:
1. Вывод `SELECT version();`
2. Удается ли создать таблицу
3. Видны ли таблицы в веб-интерфейсе на вкладке "Tables"
   
