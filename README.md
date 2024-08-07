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
5. Access oracle container environment  
   `docker exec -it oracle bash`  
   Run this bash script
   `Create sample database`
6. Check if the database was created successfully  
   `sqlplus Debezium/dbz@localhost:1521/orclpdb1`
   SELECT \* FROM INVENTORY;
