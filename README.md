# Настройка ClickHouse с репликацией и ZooKeeper

## 1. Добавить venv
Создайте виртуальное окружение:

```bash
python3 -m venv venv
source venv/bin/activate
```
## 2. Запустить контейнеры
```
docker-compose up -d
```
## 3. Подключиться к контейнеру clickhouse1
```
docker exec -it clickhouse1 bash
```
## 3.1. Создать файл zookeeper.xml
```
echo '<yandex>
    <zookeeper>
        <node>
            <host>zookeeper</host>
            <port>2181</port>
        </node>
    </zookeeper>
</yandex>' > /etc/clickhouse-server/config.d/zookeeper.xml
```
## 3.2. Добавить строку в zoo.cfg
Откройте файл конфигурации ZooKeeper:
```
nano /conf/zoo.cfg
```
Добавьте строку:
```
4lw.commands.whitelist=stat,ruok,conf
```

## 4. Настроить конфигурацию ClickHouse
Откройте файл конфигурации ClickHouse:
```
nano /etc/clickhouse-server/config.xml
```
## 4.1. Добавить подключение к ZooKeeper
Добавьте строку в заголовок документа:
```
<include_from>/etc/clickhouse-server/config.d/zookeeper.xml</include_from>
```
## 4.2. Настроить макросы и кластер
Добавьте следующие настройки:
```
<macros>
    <shard>shard1</shard>
    <replica>replica1</replica>
</macros>

<!-- Настройка ZooKeeper -->
<zookeeper>
    <node>
        <host>zookeeper</host>
        <port>2181</port>
    </node>
</zookeeper>

<!-- Настройка кластера -->
<remote_servers>
    <my_cluster>  <!-- Имя кластера -->
        <shard>  <!-- Шард 1 -->
            <replica>  <!-- Реплика 1 -->
                <host>clickhouse1</host>
                <port>9000</port>
            </replica>
            <replica>  <!-- Реплика 2 -->
                <host>clickhouse2</host>
                <port>9000</port>
            </replica>
            <replica>  <!-- Реплика 3 -->
                <host>clickhouse3</host>
                <port>9000</port>
            </replica>
        </shard>
    </my_cluster>
</remote_servers>
```

## 5. Повторить шаги 3–4 для других реплик
Для каждой реплики (clickhouse2, clickhouse3) выполните шаги 3–4, изменив только значение <replica> в макросах:

- Для clickhouse2: <replica>replica2</replica>
- Для clickhouse3: <replica>replica3</replica>

## 6. Создать сети Docker
Создайте сеть для контейнеров:
```
docker network create clickhouse_net
```
## 7. Создать базы данных
Создайте базу данных на всех репликах:
```
CREATE DATABASE IF NOT EXISTS test_database;
```
## 8. Создать реплицируемые таблицы
Создайте таблицу на каждой реплике:
```
CREATE TABLE test_database.test_table
(
    id UInt32,
    name LowCardinality(String),
    date Date
) ENGINE = ReplicatedMergeTree('/test_database/test_table', '{replica}')
ORDER BY date;
```
