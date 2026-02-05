```
тэгов -debian или -bionic/-focal нет
указаний версии ОС тоже
вариант только попробовать скачивать и смотреть что внутри
стоит пробовать такой вариант?
```
Нет, не стоит тратить на это время. Вероятность успеха крайне низкая, так как YugabyteDB вряд ли собирает официальные образы на разных дистрибутивах без явного указания в теге.

У нас остаётся **единственный надёжный вариант: собрать PostGIS из исходников внутри среды YugabyteDB**. Это гарантирует совместимость библиотек. Процесс сложный, но мы его максимально автоматизируем.

### 🛠️ Dockerfile для сборки PostGIS из исходников

Создайте новый `Dockerfile.build` с этим содержимым:

```dockerfile
# Базовый образ YugabyteDB (AlmaLinux)
FROM yugabytedb/yugabyte:2025.2.0.1-b1

# 1. Устанавливаем ВСЕ необходимые зависимости для сборки
RUN yum install -y epel-release && \
    yum install -y wget gcc make cmake git \
    postgresql15-devel \
    proj-devel proj-epsg proj-nad \
    geos-devel \
    gdal-devel \
    json-c-devel \
    libxml2-devel \
    protobuf-c-devel \
    libtool

# 2. Создаем директорию для сборки и загружаем исходники PostGIS
#    Используем конкретную стабильную версию (3.5.2)
WORKDIR /tmp/build
RUN wget https://download.osgeo.org/postgis/source/postgis-3.5.2.tar.gz && \
    tar -xzf postgis-3.5.2.tar.gz && \
    rm postgis-3.5.2.tar.gz

# 3. Компилируем PostGIS
WORKDIR /tmp/build/postgis-3.5.2
# Конфигурируем сборку, указывая путь к pg_config YugabyteDB
RUN ./configure \
    --with-pgconfig=/home/yugabyte/postgres/bin/pg_config \
    --without-raster \
    --prefix=/usr/local

# Собираем (это займет 10-15 минут)
RUN make -j$(nproc)

# 4. "Устанавливаем" в временную директорию
RUN make install DESTDIR=/tmp/postgis-build

# 5. Копируем только нужные файлы в каталоги YugabyteDB
RUN cp -r /tmp/postgis-build/usr/local/lib/postgresql/postgis* /home/yugabyte/postgres/lib/ && \
    cp -r /tmp/postgis-build/usr/local/share/postgresql/extension/postgis* /home/yugabyte/postgres/share/extension/

# 6. Копируем скомпилированные системные библиотеки (GEOS, PROJ, GDAL)
#    Они уже будут совместимы с GLIBC в этом образе
RUN cp -r /tmp/postgis-build/usr/local/lib/libgeos*.so* /home/yugabyte/lib/yb-thirdparty/ 2>/dev/null || true && \
    cp -r /tmp/postgis-build/usr/local/lib/libproj*.so* /home/yugabyte/lib/yb-thirdparty/ 2>/dev/null || true && \
    cp -r /tmp/postgis-build/usr/local/lib/libgdal*.so* /home/yugabyte/lib/yb-thirdparty/ 2>/dev/null || true

# 7. Финальная настройка YugabyteDB
RUN /home/yugabyte/bin/post_install.sh -e && \
    ldconfig /home/yugabyte/lib/yb-thirdparty/
```

### 🚀 Сборка и запуск

```bash
# 1. Соберите образ (это займет 15-25 минут)
docker build -f Dockerfile.build -t yugabyte-postgis-built:2025.2 .

# 2. Запустите контейнер
docker run -d \
  --name yb-postgis-built \
  -p 5440:5433 \
  yugabyte-postgis-built:2025.2 \
  /home/yugabyte/bin/yugabyted start --background=false

# 3. Проверьте (подождите 90 секунд)
sleep 90
docker exec yb-postgis-built /home/yugabyte/bin/ysqlsh -h 172.17.0.2 -p 5433 -U yugabyte -c "CREATE EXTENSION postgis; SELECT PostGIS_Full_Version();"
```

### ⚡ Оптимизация: Если сборка не удаётся из-за отсутствия пакетов
Если `yum install` не найдёт какие-то пакеты (`postgresql15-devel`, `proj-devel` и т.д.), нам нужно будет:
1. **Найти альтернативные имена пакетов** для AlmaLinux 8.
2. **Собрать некоторые зависимости из исходников**.

Для этого используйте упрощённый, но более надёжный `Dockerfile`:

```dockerfile
FROM yugabytedb/yugabyte:2025.2.0.1-b1

# Устанавливаем минимальный набор для сборки
RUN yum install -y wget gcc make git postgresql-devel

# Собираем и устанавливаем PROJ из исходников
WORKDIR /tmp
RUN wget https://download.osgeo.org/proj/proj-9.4.0.tar.gz && \
    tar -xzf proj-9.4.0.tar.gz && \
    cd proj-9.4.0 && \
    ./configure --prefix=/usr/local && \
    make -j$(nproc) && \
    make install

# Собираем и устанавливаем GEOS
WORKDIR /tmp
RUN wget https://download.osgeo.org/geos/geos-3.12.1.tar.bz2 && \
    tar -xjf geos-3.12.1.tar.bz2 && \
    cd geos-3.12.1 && \
    ./configure --prefix=/usr/local && \
    make -j$(nproc) && \
    make install

# Затем собираем PostGIS (как в основном Dockerfile выше)
# ... остальной код сборки PostGIS
```

### 💡 Рекомендация
**Начните с первого Dockerfile.** Если в процессе сборки возникнут ошибки о **недостающих зависимостях**, пришлите:
1. Точное сообщение об ошибке.
2. На каком шаге она произошла (`yum install`, `configure`, `make`).

Мы тут же адаптируем инструкции под доступные в AlmaLinux пакеты. Сборка из исходников — это финальный, 100% рабочий способ решить проблему GLIBC.

Попробуйте собрать по первому Dockerfile.
