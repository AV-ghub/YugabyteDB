**Отличный вопрос!** Вы правильно понимаете проблему — HAProxy сам становится единой точкой отказа. Давайте построим **полную отказоустойчивую архитектуру**.

### 🏗️ Полная схема отказоустойчивого кластера YugabyteDB

Для production-окружения нужна **многоуровневая отказоустойчивость**:

```
Ваше приложение
       ↓
[ Слой балансировки нагрузки ]
       ↓
[ YugabyteDB TServer (x3) ] ← Репликация данных
       ↓  
[ YugabyteDB Master (x3) ]  ← Управление метаданными
```

### 🔧 Решение 1: YugabyteDB Smart Driver (рекомендуется)

**Smart Driver** — это "умные" драйверы PostgreSQL, которые:
- **Знают все адреса TServer**
- **Автоматически переключаются** при сбоях
- **Распределяют нагрузку** между узлами
- **Не требуют внешнего балансировщика**

Пример подключения через Smart Driver (Java):

```java
String ybUrls = "jdbc:yugabytedb://yb-tserver1:5433,yb-tserver2:5433,yb-tserver3:5433/yugabyte";
Connection conn = DriverManager.getConnection(ybUrls, "yugabyte", "");
```

### 🔧 Решение 2: Кластеризованный HAProxy

Создайте `docker-compose.yml` с HAProxy в режиме актив-актив:

```yaml
version: '3.8'

services:
  # YugabyteDB Cluster (как у вас)
  yb-master1: &master-template
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-master"
      - "--fs_data_dirs=/mnt/master"
      - "--master_addresses=yb-master1:7100,yb-master2:7100,yb-master3:7100"
      - "--rpc_bind_addresses=yb-master1:7100"
    mem_limit: 1024m

  yb-master2:
    <<: *master-template
    command:
      - "/home/yugabyte/bin/yb-master"
      - "--fs_data_dirs=/mnt/master"
      - "--master_addresses=yb-master1:7100,yb-master2:7100,yb-master3:7100"
      - "--rpc_bind_addresses=yb-master2:7100"

  yb-master3:
    <<: *master-template
    command:
      - "/home/yugabyte/bin/yb-master"
      - "--fs_data_dirs=/mnt/master"
      - "--master_addresses=yb-master1:7100,yb-master2:7100,yb-master3:7100"
      - "--rpc_bind_addresses=yb-master3:7100"

  # TServer с разными портами
  yb-tserver1: &tserver-template
    image: yugabytedb/yugabyte:latest
    command:
      - "/home/yugabyte/bin/yb-tserver"
      - "--fs_data_dirs=/mnt/tserver"
      - "--tserver_master_addrs=yb-master1:7100,yb-master2:7100,yb-master3:7100"
      - "--rpc_bind_addresses=0.0.0.0:9100"
      - "--enable_ysql"
      - "--ysql_enable_auth=false"
      - "--pgsql_proxy_bind_address=0.0.0.0:5433"
    mem_limit: 1536m

  yb-tserver2:
    <<: *tserver-template
    ports:
      - "5434:5433"  # Разные порты наружу

  yb-tserver3:
    <<: *tserver-template
    ports:
      - "5435:5433"

  # Два HAProxy для отказоустойчивости
  haproxy1:
    image: haproxy:latest
    ports:
      - "5433:5433"  # Основной порт
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    depends_on:
      - yb-tserver1
      - yb-tserver2
      - yb-tserver3

  haproxy2:
    image: haproxy:latest
    ports:
      - "5436:5433"  # Резервный порт
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    depends_on:
      - yb-tserver1
      - yb-tserver2
      - yb-tserver3

  # Keepalived для виртуального IP (опционально)
  keepalived:
    image: osixia/keepalived:latest
    cap_add:
      - NET_ADMIN
      - NET_BROADCAST
      - NET_RAW
    volumes:
      - ./keepalived.conf:/container/service/keepalived/assets/keepalived.conf
    network_mode: "host"  # Требует особых прав
```

### 📝 Конфигурация HAProxy (haproxy.cfg)

```cfg
global
    maxconn 100

defaults
    log global
    mode tcp
    timeout connect 5s
    timeout client 50s
    timeout server 50s

# Фронтенд для подключений приложений
frontend pg_front
    bind *:5433
    default_backend pg_back

# Бэкенд с TServer узлами
backend pg_back
    balance roundrobin
    option tcp-check
    
    server yb-tserver1 yb-tserver1:5433 check port 5433
    server yb-tserver2 yb-tserver2:5433 check port 5433
    server yb-tserver3 yb-tserver3:5433 check port 5433
```

### 🔧 Решение 3: Используем встроенные возможности Kubernetes

Если развертываете в Kubernetes:

```yaml
# StatefulSet для TServer
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: yb-tserver
spec:
  serviceName: yb-tserver
  replicas: 3
  selector:
    matchLabels:
      app: yb-tserver
  template:
    metadata:
      labels:
        app: yb-tserver
    spec:
      containers:
      - name: yb-tserver
        image: yugabytedb/yugabyte:latest
        command: ["/home/yugabyte/bin/yb-tserver"]
        args:
          - "--tserver_master_addrs=$(YB_MASTER_SERVICE_HOST):7100"
          - "--enable_ysql"
          - "--pgsql_proxy_bind_address=0.0.0.0:5433"

---
# Service для балансировки
apiVersion: v1
kind: Service
metadata:
  name: yb-tserver-service
spec:
  selector:
    app: yb-tserver
  ports:
  - port: 5433
    targetPort: 5433
  type: LoadBalancer  # Или ClusterIP + внешний балансировщик
```

### 🎯 Рекомендации для разных сценариев

| Сценарий | Решение | Плюсы | Минусы |
|----------|---------|-------|--------|
| **Локальное тестирование** | Smart Driver | Простота, нет зависимостей | Нужны "умные" драйверы |
| **On-premise развертывание** | HAProxy x2 + Keepalived | Полная отказоустойчивость | Сложность настройки |
| **Cloud (AWS/Azure/GCP)** | Cloud Load Balancer + Smart Driver | Интеграция с облаком | Зависит от облачного провайдера |
| **Kubernetes** | Service LoadBalancer + Smart Driver | Нативная интеграция | Требует Kubernetes |

### ✅ Практический план для вас

1. **Продолжайте с текущей настройкой** для изучения — она уже показывает отказоустойчивость данных
2. **Используйте Smart Driver** для отказоустойчивых подключений
3. **Позже настройте HAProxy** для полной картины

**Хотите настроить Smart Driver или поэкспериментировать с HAProxy?** Или может сначала изучить другие аспекты архитектуры YugabyteDB?
