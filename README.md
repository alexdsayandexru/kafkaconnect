# kafkaconnect
Установка компонентов тестовой среды (zookeeper, kafka, schema-registry, kafka-ui,debezium)


Создать docker-compose.yml файл


version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-server:latest
    hostname: kafka
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "9997:9997"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_CONFLUENT_LICENSE_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_CONFLUENT_BALANCER_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_JMX_PORT: 9997
      KAFKA_JMX_HOSTNAME: kafka

  debezium:
    image: debezium/connect:3.0.0.Final
    environment:
      BOOTSTRAP_SERVERS: PLAINTEXT://kafka:29092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: connect_configs
      OFFSET_STORAGE_TOPIC: connect_offsets
      INTERNAL_KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      INTERNAL_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081

    depends_on: [kafka]
    ports:
      - 8083:8083

  schema-registry:
    image: confluentinc/cp-schema-registry:5.5.3
    environment:
      - SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL=zookeeper:2181
      - SCHEMA_REGISTRY_HOST_NAME=schema-registry
      - SCHEMA_REGISTRY_LISTENERS=http://schema-registry:8081,http://localhost:8081
    ports:
      - 8081:8081
    depends_on: [zookeeper, kafka]
    
  kafka-ui:
    container_name: kafka-ui
    image: provectuslabs/kafka-ui:latest
    ports:
      - 8082:8080
    environment:
      DYNAMIC_CONFIG_ENABLED: true



2. Выполнить команду в консоли: docker compose up

3. В результате должны запуститься следующие сервисы:

Настройка PostgreSQL
Установить флаг логической репликации:
ALTER SYSTEM SET wal_level = logical;
Проверить флаг логической репликации:

SELECT setting, enumvals from pg_settings WHERE name = 'wal_level';
Перезапустить сервер:

sudo -u postgres /Library/PostgreSQL/17/bin/pg_ctl -D /Library/PostgreSQL/17/data restart
Создать слот репликации в базе данных idp:

SELECT pg_create_logical_replication_slot('postgres_debezium', 'pgoutput');


Создать слот репликации в базе данных idp2

SELECT pg_create_logical_replication_slot('postgres_debezium2', 'pgoutput');
Проверить наличие слотов логической репликации:

SELECT * FROM pg_replication_slots;


Для реплицируемых таблиц устанавливаем форму информации записываемую в WAL:
ALTER TABLE users REPLICA IDENTITY FULL; // В WAL будут записываться строки со старыми значениями колонок
Возможные варианты (https://postgrespro.ru/docs/postgresql/9.4/sql-altertable):
ALTER TABLE ... REPLICA IDENTITY DEFAULT;
ALTER TABLE ... REPLICA IDENTITY USING INDEX;
ALTER TABLE ... REPLICA IDENTITY NOTHING;


Создать публикацию для реплицируемых таблиц:

CREATE PUBLICATION dbz_publication FOR TABLE users, ..., ...;
Изменить параметры публикации можно командой:

ALTER PUBLICATION dbz_publication SET (publish = 'insert, update, delete');
Удалить публикацию можно командой:

DROP PUBLICATION dbz_publication;
Просмотреть публикации и таблицы можно командой:

SELECT * FROM pg_publication;
SELECT * FROM pg_publication_tables
Подключение коннекторов (через Postman)
Подключить SOURCE коннектор можно POST запросом: http://localhost:8083/connectors
Выполняет чтение данных из WAL таблицы public.users базы данных idp и записывает в топик KAFKA idp.public.users 
Body:

{
    "name": "source-idp-postgresql-connector",
    "config":
    {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "database.hostname": "host.docker.internal",
        "database.port": "5434",
        "database.user": "postgres",
        "database.password": "postgres",
        "database.dbname": "idp",
        "plugin.name": "pgoutput",
        "database.server.name": "idp",
        "key.converter.schemas.enable": "true",
        "value.converter.schemas.enable": "true",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "table.include.list": "public.users",
        "slot.name" : "postgres_debezium2",
        "topic.prefix": "idp",
        "skip.messages.without.change": "true",
        "transforms": "unwrap",
        "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState"
    }
}
Подключить SOURCE коннектор можно POST запросом: http://localhost:8083/connectors
(Выполняет чтение данных из WAL таблицы public.users базы данных idp2 и записывает в топик KAFKA idp2.public.users)
Body:

{
    "name": "source-idp2-postgresql-connector",
    "config":
    {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "database.hostname": "host.docker.internal",
        "database.port": "5434",
        "database.user": "postgres",
        "database.password": "postgres",
        "database.dbname": "idp2",
        "plugin.name": "pgoutput",
        "database.server.name": "idp2",
        "key.converter.schemas.enable": "true",
        "value.converter.schemas.enable": "true",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "table.include.list": "public.users",
        "slot.name" : "postgres_debezium2",
        "topic.prefix": "idp2",
        "skip.messages.without.change": "true",
        "transforms": "unwrap",
        "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState"
    }
}
Подключить SYNC коннектор можно POST запросом: http://localhost:8083/connectors
(Выполняет чтение данных из топика KAFKA idp.public.users и записывает в таблицу public.users базы данных idp)
Body:

{
    "name": "sync-idp-jdbc-connector",  
    "config": 
    {
        "connector.class": "io.debezium.connector.jdbc.JdbcSinkConnector",  
        "tasks.max": "1",  
        "connection.url": "jdbc:postgresql://host.docker.internal:5434/idp2",  
        "connection.username": "postgres",  
        "connection.password": "postgres",  
        "insert.mode": "upsert",   
        "primary.key.mode": "record_key",
        "schema.evolution": "basic",  
        "topics": "idp.public.users",
        "hibernate.dialect": "org.hibernate.dialect.PostgreSQLDialect",
        "schemas.enable": "true",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "auto.create": "true",
        "auto.evolve": "true",
        "transforms": "dropTopicPrefix",
        "transforms.dropTopicPrefix.type":"org.apache.kafka.connect.transforms.RegexRouter",
        "transforms.dropTopicPrefix.regex":"^[^.]+.[^.]++.(.*)",
        "transforms.dropTopicPrefix.replacement":"$1",
        "errors.tolerance": "all",
        "errors.deadletterqueue.topic.name": "myDLQTopicName",
        "errors.log.include.messages": "true",
        "errors.log.enable": "true"
    }
}
Подключить SYNC коннектор можно POST запросом: http://localhost:8083/connectors
(Выполняет чтение данных из топика KAFKA idp2.public.users и записывает в таблицу public.users базы данных idp2)
Body:

{
    "name": "sync-idp2-jdbc-connector",
    "config":
    {
        "connector.class": "io.debezium.connector.jdbc.JdbcSinkConnector",
        "tasks.max": "1",
        "connection.url": "jdbc:postgresql://host.docker.internal:5434/idp",
        "connection.username": "postgres",
        "connection.password": "postgres",
        "insert.mode": "upsert",
        "primary.key.mode": "record_key",
        "schema.evolution": "basic",
        "topics": "idp2.public.users",
        "hibernate.dialect": "org.hibernate.dialect.PostgreSQLDialect",
        "schemas.enable": "true",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "auto.create": "true",
        "auto.evolve": "true",
        "transforms": "dropTopicPrefix",
        "transforms.dropTopicPrefix.type":"org.apache.kafka.connect.transforms.RegexRouter",
        "transforms.dropTopicPrefix.regex":"^[^.]+.[^.]++.(.*)",
        "transforms.dropTopicPrefix.replacement":"$1",
        "errors.tolerance": "all",
        "errors.deadletterqueue.topic.name": "myDLQTopicName2",
        "errors.log.include.messages": "true",
        "errors.log.enable": "true"
    }
}
Просмотреть список подключенных коннекторов можно GET запросом: http://localhost:8083/connectors


Результат:
[
"source-idp-postgresql-connector",
"sync-idp-jdbc-connector",
"source-idp2-postgresql-connector",
"sync-idp2-jdbc-connector"
]

Удалить коннектор можно DELETE запросом: http://localhost:8083/connectors/source-idp2-postgresql-connector
Просмотреть статус коннектора можно GET запросом: http://localhost:8083/connectors/sync-idp2-jdbc-connector/status
Результат:

{
    "name": "sync-idp2-jdbc-connector",
    "connector": {
        "state": "RUNNING",
        "worker_id": "172.27.0.5:8083"
    },
    "tasks": [
        {
            "id": 0,
            "state": "RUNNING",
            "worker_id": "172.27.0.5:8083"
        }
    ],
    "type": "sink"
}


Ссылки на материалы
https://habr.com/ru/companies/flant/articles/523510/
https://habr.com/ru/companies/first/articles/668516/
https://github.com/debezium/debezium-examples/tree/main/tutorial
