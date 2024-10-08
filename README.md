# Oracle CDC to Data Lake using Kafka Confluent

This project demonstrates a data engineering pipeline that performs Change Data Capture (CDC) from an Oracle database to a data lake using Apache Kafka and Confluent, with Apache Iceberg as the table format and MinIO as the object storage.

## Architecture

This pipeline captures real-time changes from an Oracle database and streams them to a data lake, enabling efficient data replication and analytics.

1. Oracle Database (Source)
2. Debezium Connector for Oracle
3. Apache Kafka (Confluent Platform)
4. Kafka Connect Iceberg Sink Connector
5. Apache Iceberg
6. MinIO (S3-compatible object storage)

## Key Features

- Real-time data capture from Oracle using Debezium
- Scalable and fault-tolerant data streaming with Kafka
- Efficient data lake storage using Apache Iceberg table format
- S3-compatible object storage with MinIO

## Prerequisites

- Docker and Docker Compose

## Setup and Installation

### 1. Setup source: Oracle

Pull image from Oracle image registry: https://container-registry.oracle.com/ords/ocr/ba/database/enterprise

1. Create an Oracle container registry account
2. Sign in and accept the license agreement
3. Login to oracle registry in the local environment, using this command: <br>
   `docker login container-registry.oracle.com`
4. Pull image version 21c <br>
   `docker pull container-registry.oracle.com/database/enterprise:21.3.0.0`
5. Start up containers
   `docker compose up -d`
6. Access oracle container environment  
   `docker exec -it oracle bash`  
   Run this bash script to setup logminer  
   `curl https://raw.githubusercontent.com/thanhnh-de/oracle-cdc-to-data-lake/main/oracle/setup_logminer.sh | sh`  
   Run this sql script to initialize database  
   `curl https://raw.githubusercontent.com/thanhnh-de/oracle-cdc-to-data-lake/main/oracle/setup_db.sql | sqlplus debezium/dbz@//localhost:1521/orclpdb1`
7. Check if the database was created successfully  
   `sqlplus Debezium/dbz@localhost:1521/orclpdb1`  
   `SELECT \* FROM INVENTORY;`

### 2. Install kafka connect (source and sink) plugin

1. Enter kafka connect container  
   `docker exec -it -u root connect bash`
2. Navigate to CONNECT_PLUGIN_PATH (where plugins are installed)  
   `cd /usr/share/java`

- Download Oracle connector plugin archive  
  `wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-oracle/2.7.0.Final/debezium-connector-oracle-2.7.0.Final-plugin.tar.gz`
- Extract the archive  
  `tar -xvzf debezium-connector-oracle-2.7.0.Final-plugin.tar.gz`
- Download JDBC driver  
  `wget https://repo1.maven.org/maven2/com/oracle/database/jdbc/ojdbc8/21.11.0.0/ojdbc8-21.11.0.0.jar`
- Move the driver to extracted connector plugin folder  
  `mv ojdbc8-21.11.0.0.jar debezium-connector-oracle`
- Remove the archive file  
  `rm -f debezium-connector-oracle-2.7.0.Final-plugin.tar.gz`
- Do the same with iceberg sink connector:  
  `yum -y install zip`  
  `wget https://github.com/tabular-io/iceberg-kafka-connect/releases/download/v0.6.19/iceberg-kafka-connect-runtime-0.6.19.zip`  
  `unzip iceberg-kafka-connect-runtime-0.6.19.zip`  
  `rm -f iceberg-kafka-connect-runtime-0.6.19.zip`
- Exit container  
  `exit`
- Restart connect container to fetch previously installed plugin  
  `docker restart connect`  
  then wait until the container finishes starting up

### 3. Setup source connector

Download this connector file to local environemnt: connector_config/source_oracle.json

- Access confluent Control center at: http://localhost:9021/
- Choose controlcenter.cluster
- Navigate to Connect -> Choose connect-default cluster -> Select "Add connector" -> "Upload connector config file"
- Upload the downloaded file
- Scroll down to the end of the page -> click Next -> Launch
- After a while, we can check the created topic at section Topic (test.DEBEZIUM.INVENTORY)

### 4. Transform topic message with ksqlDB

- On the control center interface, navigate to ksqlDB section
- Set auto.offset.rest to Earliest
- Create stream for the above topic:

```
CREATE STREAM INVENTORY_CHANGES WITH (
  KAFKA_TOPIC = 'test.DEBEZIUM.INVENTORY',
  KEY_FORMAT = 'AVRO',
  VALUE_FORMAT = 'AVRO'
);
```

- Click Run Query
- Check the stream: `select * from  INVENTORY_CHANGES;`
- Manipulate the stream by transforming and enriching it:

```
CREATE STREAM INVENTORY_CHANGES_PROCESSED AS
SELECT
  rowkey,
  COALESCE(before->ITEM_NAME, after->ITEM_NAME) as ITEM_NAME,
  COALESCE(before->DETAILS, after->DETAILS) as DETAILS,
  COALESCE(before->PRICE, after->PRICE) as PRICE,
  op as op_type,
  ts_ms as op_ts,
  source->scn as scn,
  source->`CONNECTOR` as `connector`,
  source->db + '.' + source->schema + '.' + source->`TABLE` as `table`
FROM INVENTORY_CHANGES
EMIT CHANGES;
```

After doing this, you'll see a newly created topic in the topic panel, now we will sink this topic to our data lake

### 5. Setup sink connector

- Upload file connector_config/sink_iceberg.json to confluent platform
- After a while, check the sinked data in the data lake
  `docker exec -it spark-iceberg spark-sql`
  `SELECT * FROM psa.inventory`
- You'll see the data has landed in the data lake area with some additional CDC fields, which can be useful in the downstream processes  
  **Note**: the column price is originally float and is converted to type STRUCT<SCALE: INT, VALUE: BINARY>. To convert it to readable format, use this query:  
  `select CAST(conv(hex(price.VALUE), 16, 10) AS DOUBLE) / POW(10, price.SCALE) AS float_value from psa.inventory`
