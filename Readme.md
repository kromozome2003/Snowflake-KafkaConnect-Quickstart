# Snowflake-KafkaConnect-Quickstart
Consuming Kafka messages from Confluent and loading messages payload into a table (per topic)

Online documentation : https://docs.snowflake.net/manuals/user-guide/kafka-connector.html

## Prerequisites
* Docker version 1.11 or later is installed and running.
* Docker Compose is installed. Docker Compose is installed by default with Docker for Mac.
* Docker memory is allocated minimally at 8 GB. When using Docker Desktop for Mac, the default Docker memory allocation is 2 GB. You can change the default allocation to 8 GB in Docker > Preferences > Advanced.
* Git.
* Internet connectivity.
* Ensure you are on an Operating System currently supported by Confluent Platform.
* Networking and Kafka on Docker: Configure your hosts and ports to allow both internal and external components to the Docker network to communicate. For more details, see this article.

## Introduction
Kafka Connect is a framework for connecting Kafka with external systems, including databases. A Kafka Connect cluster is a separate cluster from the Kafka cluster. The Kafka Connect cluster supports running and scaling out connectors (components that support reading and/or writing between external systems).

The Kafka connector is designed to run in a Kafka Connect cluster to read data from Kafka topics and write the data into Snowflake tables. Snowflake provides two versions of the connector:

* A version for the Confluentpackage version of Kafka.
For more information about Kafka Connect, see https://docs.confluent.io/3.0.0/connect/.

* A version for the open source software (OSS) Apache Kafka package.
For more information about Apache Kafka, see https://kafka.apache.org/.

From the perspective of Snowflake, a Kafka topic produces a stream of rows to be inserted into a Snowflake table. In general, each Kafka message contains one row.

Kafka, like many message publish/subscribe platforms, allows a many-to-many relationship between publishers and subscribers. A single application can publish to many topics, and a single application can subscribe to multiple topics. With Snowflake, the typical pattern is that one topic supplies messages (rows) for one Snowflake table.

## What's in this tutorial ?
In this tutorial we are going to deploy a Confluent Kafka cluster, setup the Snowflake Kafka Connector and Generate dummy messages every 100ms that will be automatically consumed and inserted into a Snowflake table without custom code.

![](/images/sf-kafkaconnect-diagram.png)

## To begin
### Retrieve this github repo
```
git clone https://github.com/kromozome2003/Snowflake-KafkaConnect-Quickstart.git
cd Snowflake-KafkaConnect-Quickstart
```

## Prepare a key pair (required by KafkaConnect)
### A more detailed tuto is available [here](https://docs.snowflake.net/manuals/user-guide/kafka-connector-install.html#using-key-pair-authentication)
First, generate an encrypted private key with aes256 (with a passphrase)
```
openssl genrsa 2048 | openssl pkcs8 -topk8 -v2 aes256 -inform PEM -out rsa_key.p8
```
below is the file where your private key has been stored (to use in Confluent & Snowsql)
```
cat rsa_key.p8 | awk ' !/-----/ {printf $1}'
```
Then, generate a public key based on the private one
```
openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub
```
below is the file where your public key has been stored (to associate w/Snowflake user : RSA_PUBLIC_KEY)
```
cat rsa_key.pub | awk '!/-----/{printf $1}'
```

## Snowflake Initial Setup
In order to be able to consume messages from our Kafka Connector you need to creates some objects.
A more detailed procedure is available [Here](https://docs.snowflake.net/manuals/user-guide/kafka-connector-install.html)
### Create a Snowflake DB
```
DROP DATABASE IF EXISTS KAFKA_DB;
CREATE OR REPLACE DATABASE KAFKA_DB COMMENT = 'Database for KafkaConnect demo';
```

### Create a Snowflake ROLE
```
USE ROLE SECURITYADMIN;
DROP ROLE IF EXISTS KAFKA_CONNECTOR_ROLE;
CREATE ROLE KAFKA_CONNECTOR_ROLE;
GRANT ALL ON DATABASE KAFKA_DB TO ROLE KAFKA_CONNECTOR_ROLE;
GRANT ALL ON DATABASE KAFKA_DB TO ACCOUNTADMIN;
GRANT ALL ON SCHEMA KAFKA_DB.PUBLIC TO ROLE KAFKA_CONNECTOR_ROLE;
GRANT ALL ON FUTURE TABLES IN SCHEMA KAFKA_DB.PUBLIC TO KAFKA_CONNECTOR_ROLE;
GRANT ALL ON SCHEMA KAFKA_DB.PUBLIC TO ROLE ACCOUNTADMIN;
GRANT CREATE TABLE ON SCHEMA KAFKA_DB.PUBLIC TO ROLE KAFKA_CONNECTOR_ROLE;
GRANT CREATE STAGE ON SCHEMA KAFKA_DB.PUBLIC TO ROLE KAFKA_CONNECTOR_ROLE;
GRANT CREATE PIPE ON SCHEMA KAFKA_DB.PUBLIC TO ROLE KAFKA_CONNECTOR_ROLE;
```
### Create a Snowflake WAREHOUSE (for admin purpose as KafkaConnect is Serverless)
```
USE ROLE ACCOUNTADMIN;
CREATE OR REPLACE WAREHOUSE KAFKA_ADMIN_WAREHOUSE
  WAREHOUSE_SIZE = 'XSMALL'
  WAREHOUSE_TYPE = 'STANDARD'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD'
  INITIALLY_SUSPENDED = TRUE
  COMMENT = 'Warehouse for Kafka admin activities';
GRANT USAGE ON WAREHOUSE KAFKA_ADMIN_WAREHOUSE TO ROLE KAFKA_CONNECTOR_ROLE;
```

### Create a Snowflake USER
```
USE ROLE ACCOUNTADMIN;
DROP USER IF EXISTS KAFKA_DEMO;
CREATE USER KAFKA_DEMO
 PASSWORD = 'Password123!'
 LOGIN_NAME = KAFKA_DEMO
 DISPLAY_NAME = KAFKA_DEMO
 DEFAULT_WAREHOUSE = KAFKA_ADMIN_WAREHOUSE
 DEFAULT_ROLE = KAFKA_CONNECTOR_ROLE
 DEFAULT_NAMESPACE = KAFKA_DB
 MUST_CHANGE_PASSWORD = FALSE
 RSA_PUBLIC_KEY="<your_private_key>";
GRANT ROLE KAFKA_CONNECTOR_ROLE TO USER KAFKA_DEMO;
```

### Create some views for Tableau dashboards
```
USE ROLE KAFKA_CONNECTOR_ROLE;
USE DATABASE KAFKA_DB;
USE SCHEMA PUBLIC;
USE WAREHOUSE KAFKA_ADMIN_WAREHOUSE;

-- Clickstream view from AVRO records
CREATE OR REPLACE SECURE VIEW CLICKSTREAM_VW AS
SELECT
    RECORD_CONTENT:_time::string _TIME,
    RECORD_CONTENT:agent::string AGENT,
    RECORD_CONTENT:bytes::string BYTES,
    RECORD_CONTENT:ip::string IP,
    RECORD_CONTENT:referrer::string REFERRER,
    RECORD_CONTENT:remote_user::string REMOTE_USER,
    RECORD_CONTENT:request::string REQUEST,
    RECORD_CONTENT:status::string STATUS,
    RECORD_CONTENT:time::string TIME,
    RECORD_CONTENT:userid::string USERID
    FROM CLICKSTREAM;

-- Pageviews view from AVRO records
CREATE OR REPLACE SECURE VIEW PAGEVIEWS_VW AS
SELECT
    RECORD_CONTENT:pageid::string PAGEID,
    RECORD_CONTENT:userid::string USERID,
    RECORD_CONTENT:viewtime::string VIEWTIME
    FROM PAGEVIEWS;

-- Orders view from AVRO records
CREATE OR REPLACE SECURE VIEW ORDERS_VW AS
SELECT
    RECORD_CONTENT:address:city::string CITY,
    RECORD_CONTENT:address:state::string STATE,
    RECORD_CONTENT:address:zipcode::int ZIPCODE,
    RECORD_CONTENT:itemid::string ITEMID,
    RECORD_CONTENT:orderid::int ORDERID,
    RECORD_CONTENT:ordertime::int ORDERTIME,
    RECORD_CONTENT:orderunits::float ORDERUNITS
    FROM ORDERS;

-- Ratings view from AVRO records
CREATE OR REPLACE SECURE VIEW RATINGS_VW AS
SELECT
    RECORD_CONTENT:channel::string CHANNEL,
    RECORD_CONTENT:message::string MESSAGE,
    RECORD_CONTENT:rating_id::int RATINGID,
    RECORD_CONTENT:rating_time::int RATINGTIME,
    RECORD_CONTENT:route_id::int ROUTEID,
    RECORD_CONTENT:stars::int STARS,
    RECORD_CONTENT:user_id::int USERID
    FROM RATINGS;

-- Stock_trades view from AVRO records
CREATE OR REPLACE SECURE VIEW STOCK_TRADES_VW AS
SELECT
    RECORD_CONTENT:account::string ACCOUNT,
    RECORD_CONTENT:price::int PRICE,
    RECORD_CONTENT:quantity::int QUANTITY,
    RECORD_CONTENT:side::string SIDE,
    RECORD_CONTENT:symbol::string SYMBOL,
    RECORD_CONTENT:userid::string USERID
    FROM STOCK_TRADES;

-- Users view from AVRO records
CREATE OR REPLACE SECURE VIEW USERS_VW AS
SELECT
    RECORD_CONTENT:gender::string GENDER,
    RECORD_CONTENT:regionid::string REGIONID,
    RECORD_CONTENT:registertime::int REGISTERTIME,
    RECORD_CONTENT:userid::string USERID
    FROM USERS;

```

### Test the setup with SnowSQL
If snowsql is not installed yet please follow this [LINK](https://docs.snowflake.net/manuals/user-guide/snowsql-install-config.html)
In my case my snowflake account is `eu_demo32.eu-central-1`
```
snowsql -a eu_demo32.eu-central-1 -u kafka_demo --private-key-path rsa_key.p8
```

## Confluent Kafka Initial Setup
A mode detailed tutorial is available [here](https://docs.confluent.io/current/quickstart/ce-docker-quickstart.html)

### In a terminal, deploy the Confluent Quickstart Containers
```
docker-compose up -d --build
docker-compose ps
```

### Confluent Kafka is now Deployed in Docker, Up & Running.
Open a web browser (Chrome) and go to [http://localhost:9021/](http://localhost:9021/)

### Let's create some 'dummy' topics (in the same terminal)
```
docker-compose exec connect bash -c 'kafka-topics --create --topic pageviews --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181'
docker-compose exec connect bash -c 'kafka-topics --create --topic stock_trades --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181'
docker-compose exec connect bash -c 'kafka-topics --create --topic clickstream --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181'
docker-compose exec connect bash -c 'kafka-topics --create --topic orders --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181'
docker-compose exec connect bash -c 'kafka-topics --create --topic ratings --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181'
docker-compose exec connect bash -c 'kafka-topics --create --topic users --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181'
```

### Let's deploy Snowflake Kafka Connector for Confluent in the `connect` container
```
docker-compose exec connect bash -c 'confluent-hub install --no-prompt --verbose snowflakeinc/snowflake-kafka-connector:1.1.0'
```

### Restart the Kafka-Connect container after the install
```
docker-compose restart connect
```
You can see the log with : `docker-compose logs connect | grep -i snowflake`

### Lets create two connectors :
* 1 connector to generate random AVRO messages (DatagenConnector)
* 1 for Snowflake Sink connector to consume & load those messages into your Snowflake KAFKA_DB (SnowflakeSinkConnector)

In the browser [http://localhost:9021/](http://localhost:9021/)
* Cluster 1 -> Connect -> `connect-default` -> Add Connector -> Upload connector config file -> `connector_snowflake_xxx.json`
Update Snowflake Login Info -> Continue -> Launch (Should see : Running)
* Cluster 1 -> Connect -> `connect-default` -> Upload connector config file -> `connector_datagen_xxx.json` -> Continue -> Launch (Should see : Running)
* Cluster 1 -> Topics -> `xxx` -> Messages (messages should appears)

### Check if messages arrives in Snowflake table (logged as KAFKA_DEMO user)
```
USE ROLE KAFKA_CONNECTOR_ROLE;
USE DATABASE KAFKA_DB;
USE SCHEMA PUBLIC;
USE WAREHOUSE KAFKA_ADMIN_WAREHOUSE;

SHOW TABLES;

-- Pageviews
SELECT count(*) FROM KAFKA_DB.PUBLIC.PAGEVIEWS;
SELECT * FROM KAFKA_DB.PUBLIC.PAGEVIEWS LIMIT 10;
SELECT * FROM TABLE(information_schema.copy_history(table_name=>'KAFKA_DB.PUBLIC.PAGEVIEWS', start_time=> dateadd(hours, -1, current_timestamp())));

-- Orders
SELECT count(*) FROM KAFKA_DB.PUBLIC.ORDERS;
SELECT * FROM KAFKA_DB.PUBLIC.ORDERS LIMIT 10;
SELECT * FROM TABLE(information_schema.copy_history(table_name=>'KAFKA_DB.PUBLIC.ORDERS', start_time=> dateadd(hours, -1, current_timestamp())));

-- Ratings
SELECT count(*) FROM KAFKA_DB.PUBLIC.RATINGS;
SELECT * FROM KAFKA_DB.PUBLIC.RATINGS LIMIT 10;
SELECT * FROM TABLE(information_schema.copy_history(table_name=>'KAFKA_DB.PUBLIC.RATINGS', start_time=> dateadd(hours, -1, current_timestamp())));

-- Stock_trades
SELECT count(*) FROM KAFKA_DB.PUBLIC.STOCK_TRADES;
SELECT * FROM KAFKA_DB.PUBLIC.STOCK_TRADES LIMIT 10;
SELECT * FROM TABLE(information_schema.copy_history(table_name=>'KAFKA_DB.PUBLIC.STOCK_TRADES', start_time=> dateadd(hours, -1, current_timestamp())));

-- Users
SELECT count(*) FROM KAFKA_DB.PUBLIC.USERS;
SELECT * FROM KAFKA_DB.PUBLIC.USERS LIMIT 10;
SELECT * FROM TABLE(information_schema.copy_history(table_name=>'KAFKA_DB.PUBLIC.USERS', start_time=> dateadd(hours, -1, current_timestamp())));

-- Clickstream
SELECT count(*) FROM KAFKA_DB.PUBLIC.CLICKSTREAM;
SELECT * FROM KAFKA_DB.PUBLIC.CLICKSTREAM LIMIT 10;
SELECT * FROM TABLE(information_schema.copy_history(table_name=>'KAFKA_DB.PUBLIC.CLICKSTREAM', start_time=> dateadd(hours, -1, current_timestamp())));

```

Mike Uzan - Senior SE (EMEA/France)
mike.uzan@snowflake.com
+33621728792
