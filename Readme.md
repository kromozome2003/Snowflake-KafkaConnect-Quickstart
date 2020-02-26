

git clone https://github.com/kromozome2003/Snowflake-KafkaConnect-Quickstart.git
cd Snowflake-KafkaConnect-Quickstart



## Prepare a key pair (required by KafkaConnect)
### A more detailed tuto is available [here](https://docs.snowflake.net/manuals/user-guide/kafka-connector-install.html#using-key-pair-authentication)
First, generate an encrypted private key with aes256 (with a passphrase)
```
openssl genrsa 2048 | openssl pkcs8 -topk8 -v2 aes256 -inform PEM -out rsa_key.p8
```
below is the file where your private key has been stored (to use in Confluent & Snowsql)
```
cat rsa_key.p8
```
Then, generate a public key based on the private one
```
openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub
```
below is the file where your public key has been stored (to associate w/Snowflake user : RSA_PU£BLIC_KEY)
```
cat rsa_key.pub
```

## Snowflake Initial Setup
In order to be able to consume messages from our Kafka Connector you need to creates some objects.
A more detailed procedure is available [Here](https://docs.snowflake.net/manuals/user-guide/kafka-connector-install.html)
### Create a Snowflake DB
```
DROP TABLE IF EXISTS KAFKA_DB;
CREATE OR REPLACE DATABASE KAFKA_DB COMMENT = 'Database for KafkaConnect demo';
```

### Create a Snowflake ROLE
```
USE ROLE SECURITYADMIN;
CREATE ROLE KAFKA_CONNECTOR_ROLE;
GRANT USAGE ON DATABASE KAFKA_DB TO ROLE KAFKA_CONNECTOR_ROLE;
GRANT USAGE ON DATABASE KAFKA_DB TO ACCOUNTADMIN;
GRANT USAGE ON SCHEMA KAFKA_DB.PUBLIC TO ROLE KAFKA_CONNECTOR_ROLE;
GRANT USAGE ON FUTURE TABLE IN SCHEMA KAFKA_DB.PUBLIC TO KAFKA_CONNECTOR_ROLE;
GRANT USAGE ON SCHEMA KAFKA_DB.PUBLIC TO ROLE ACCOUNTADMIN;
GRANT CREATE TABLE ON SCHEMA KAFKA_DB.PUBLIC TO ROLE KAFKA_CONNECTOR_ROLE;
GRANT CREATE STAGE ON SCHEMA KAFKA_DB.PUBLIC TO ROLE KAFKA_CONNECTOR_ROLE;
GRANT CREATE PIPE ON SCHEMA KAFKA_DB.PUBLIC TO ROLE KAFKA_CONNECTOR_ROLE;
```
### Create a Snowflake WAREHOUSE (for admin purpose as KafkaConnect is Serverless)
```
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

### Test the setup with SnowSQL
If snowsql is not installed yet please follow this [LINK](https://docs.snowflake.net/manuals/user-guide/snowsql-install-config.html)
In my case my snowflake account is `eu_demo32.eu-central-1`
```
snowsql -a eu_demo32.eu-central-1 -u kafka_demo --private-key-path rsa_key.p8
```

## Confluent Kafka Initial Setup
A mode detailed tutorial is available [here](https://docs.confluent.io/current/quickstart/ce-docker-quickstart.html)

### Prerequisites
* Docker version 1.11 or later is installed and running.
* Docker Compose is installed. Docker Compose is installed by default with Docker for Mac.
* Docker memory is allocated minimally at 8 GB. When using Docker Desktop for Mac, the default Docker memory allocation is 2 GB. You can change the default allocation to 8 GB in Docker > Preferences > Advanced.
* Git.
* Internet connectivity.
* Ensure you are on an Operating System currently supported by Confluent Platform.
* Networking and Kafka on Docker: Configure your hosts and ports to allow both internal and external components to the Docker network to communicate. For more details, see this article.

### In a terminal, deploy the Confluent Quickstart Containers
```
docker-compose up -d --build
docker-compose ps
```

### Confluent Kafka is now Deployed in Docker, Up & Running.
Open a web browser (Chrome) and go to [http://localhost:9021/](http://localhost:9021/)

### Let's create your first Topic `pageviews` (in the same terminal)
```
docker-compose exec connect bash -c 'kafka-topics --create --topic pageviews --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181'
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
* Cluster 1 -> Connect -> `connect-default` -> Add Connector -> Upload connector config file -> `connector_snowflake.json`
Update Snowflake Login Info -> Continue -> Launch (Should see : Running)
* Cluster 1 -> Connect -> `connect-default` -> Upload connector config file -> `connector_datagen-pageviews_config.json` -> Continue -> Launch (Should see : Running)
* Cluster 1 -> Topics -> `pageviews` -> Messages (messages should appears)

### Check if messages arrives in Snowflake table (logged as KAFKA_DEMO user)
```
USE ROLE KAFKA_CONNECTOR_ROLE;
USE DATABASE KAFKA_DB;
USE SCHEMA PUBLIC;
USE WAREHOUSE KAFKA_ADMIN_WAREHOUSE;
SELECT * FROM KAFKA_DB.PUBLIC.PAGEVIEWS;
SELECT * FROM TABLE(information_schema.copy_history(table_name=>'KAFKA.PUBLIC.PAGEVIEWS', start_time=> dateadd(hours, -1, current_timestamp())));
```

Mike Uzan - Senior SE (EMEA/France)
mike.uzan@snowflake.com
+33621728792
