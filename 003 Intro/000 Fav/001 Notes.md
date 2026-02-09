# Снаружи прописываем корпоративные сертификаты прокси
В соотв с инструкцией

# Обезвреживание SSL внутри контейнера
```
echo "sslverify=false" >> /etc/dnf/dnf.conf
echo "sslverify=0" >> /etc/yum.conf
echo "insecure=1" >> /etc/yum.conf

update-ca-trust
update-ca-trust extract

yum updtae -y
dnf updtae -y

yum install -y ca-certificates
```

# Контейнер
```
# Проверьте статус контейнера
docker ps -a | grep yb-build

# Запустите его и подключитесь
docker start yb-build

# Прямой вход в существующий контейнер
# Если вы не хотите терять уже установленные в контейнере пакеты (если они были), можно просто зайти в уже работающий контейнер.
# Для этого выполните на хостовой машине (вне контейнера):
docker exec -it yb-build /bin/bash

# Остановка контейнера
docker stop yb-build

# Удалить остановленный контейнер
docker rm yb-build

# Создать и запустить новый контейнер
docker run -it --name yb-build -v /tmp/postgis_build:/mnt yugabytedb/yugabyte:2025.2.0.1-b1-x86_64 /bin/bash

```

## Краткая памятка по командам Docker
Цель	Команда	Примечание

Остановить контейнер	          
docker stop <имя>	
Останавливает запущенный контейнер.

Удалить остановленный контейнер	
docker rm <имя>	
Удаляет контейнер (только если он остановлен).

Принудительно удалить	          
docker rm -f <имя>	
Останавливает (если запущен) и сразу удаляет.

Зайти в запущенный контейнер	  
docker exec -it <имя> /bin/bash	
Открывает оболочку внутри контейнера.

Просмотреть все контейнеры	    
docker ps -a	
Показывает все контейнеры (и запущенные, и остановленные).

```
# docker --version
Docker version 20.10.17, build 100c701
 
# docker compose --version
Docker version 20.10.17, build 100c701
 
# Действующие артефакты докера
 
# docker ps
CONTAINER ID   IMAGE                                                          COMMAND                  CREATED       STATUS       PORTS                                                     NAMES
 
# docker-compose ps
Name Command State Ports
```

##  Проверяем сети Docker
```
docker network ls
docker network inspect yugabyte_default
docker exec yugabyte_yb-tserver1_1 netstat -tln # Контейнер может слушать на своих портах
# 1. Просмотр всех контейнеров и их сетей
docker ps --format "table {{.Names}}\t{{.Ports}}\t{{.Status}}"

```

#  Проверяем параметры системы
```
cat /etc/os-release
hostnamectl
hostname -I
uname -a
netstat -tln
ss -tlnp | grep docker
```

















