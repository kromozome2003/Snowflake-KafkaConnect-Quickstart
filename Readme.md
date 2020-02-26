
# Prepare a key pair (required by KafkaConnect)
```openssl genrsa 2048 | openssl pkcs8 -topk8 -v2 aes256 -inform PEM -out rsa_key.p8```
this is your private key (to use in Confluent & Snowsql)
```cat rsa_key.p8
openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub```
this is your public key (to associate w/Snowflake user : RSA_PU£BLIC_KEY)
```cat rsa_key.pub```

# Create Snowflake DB
  ```DROP TABLE IF EXISTS KAFKA_DB;
  CREATE OR REPLACE DATABASE KAFKA_DB COMMENT = 'Database for KafkaConnect demo';```

# Create Snowflake ROLE
  ```USE ROLE SECURITYADMIN;
  CREATE ROLE KAFKA_CONNECTOR_ROLE;
  GRANT USAGE ON DATABASE KAFKA_DB TO ROLE KAFKA_CONNECTOR_ROLE;
  GRANT USAGE ON DATABASE KAFKA_DB TO ACCOUNTADMIN;
  GRANT USAGE ON SCHEMA KAFKA_DB.PUBLIC TO ROLE KAFKA_CONNECTOR_ROLE;
  GRANT USAGE ON FUTURE TABLE IN SCHEMA KAFKA_DB.PUBLIC TO KAFKA_CONNECTOR_ROLE;
  GRANT USAGE ON SCHEMA KAFKA_DB.PUBLIC TO ROLE ACCOUNTADMIN;
  GRANT CREATE TABLE ON SCHEMA KAFKA_DB.PUBLIC TO ROLE KAFKA_CONNECTOR_ROLE;
  GRANT CREATE STAGE ON SCHEMA KAFKA_DB.PUBLIC TO ROLE KAFKA_CONNECTOR_ROLE;
  GRANT CREATE PIPE ON SCHEMA KAFKA_DB.PUBLIC TO ROLE KAFKA_CONNECTOR_ROLE;```

# Create Snowflake USER
  ```USE ROLE ACCOUNTADMIN;
  DROP USER IF EXISTS KAFKA_DEMO;
  CREATE USER KAFKA_DEMO
	 PASSWORD = 'Password123!'
   LOGIN_NAME = KAFKA_DEMO
   DISPLAY_NAME = KAFKA_DEMO
   DEFAULT_ROLE = KAFKA_CONNECTOR_ROLE
   DEFAULT_NAMESPACE = KAFKA_DB
   MUST_CHANGE_PASSWORD = FALSE
   RSA_PUBLIC_KEY="<your_private_key>";
  GRANT ROLE KAFKA_CONNECTOR_ROLE TO USER KAFKA_DEMO;```

# Test the setup w/SnowSQL
  ```snowsql -a eu_demo32.eu-central-1 -u kafka_demo --private-key-path rsa_key.p8```

# Let's install [Confluent w/Docker](https://docs.confluent.io/current/quickstart/ce-docker-quickstart.html)
  ## Prerequisites
  Docker:
  * Docker version 1.11 or later is installed and running.
  * Docker Compose is installed. Docker Compose is installed by default with Docker for Mac.
  * Docker memory is allocated minimally at 8 GB. When using Docker Desktop for Mac, the default Docker memory allocation is 2 GB. You can change the default allocation to 8 GB in Docker > Preferences > Advanced.
  * Git.
  * Internet connectivity.
  * Ensure you are on an Operating System currently supported by Confluent Platform.
  * Networking and Kafka on Docker: Configure your hosts and ports to allow both internal and external components to the Docker network to communicate. For more details, see this article.

  ## Open a terminal :
  ```git clone https://github.com/confluentinc/examples
  cd examples
  git checkout 5.4.0-post
  cd cp-all-in-one/
  docker-compose up -d --build
  docker-compose ps```

# Confluent Kafka is now Deployed in Docker, Up & Running.
  Open a web browser (Chrome) and go to [http://localhost:9021/](http://localhost:9021/)

# Let's create your first Topic `pageviews` (in the same terminal)
  ```docker-compose exec connect bash -c 'kafka-topics --create --topic pageviews --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181'```

# Let's deploy Snowflake Kafka Connector for Confluent (in the same terminal)
  ```docker-compose exec connect bash -c 'confluent-hub install --no-prompt --verbose snowflakeinc/snowflake-kafka-connector:1.1.0'```
# Restart the Kafka-Connect container after the install
  ```docker-compose restart connect```
  ## You can see the log with : docker-compose logs connect | grep -i snowflake

# Lets create two connectors :
1 for generate random AVRO messages (DatagenConnector) and 1 for Snowflake to consume & load those messages into KAFKA_DB (SnowflakeSinkConnector)
  ## In the browser [http://localhost:9021/](http://localhost:9021/)
  Cluster 1 -> Connect -> `connect-default` -> Add Connector -> Upload connector config file -> `connector_snowflake.json`
    ## Update Snowflake Login Info -> Continue -> Launch (Should see : Running)
  Cluster 1 -> Connect -> `connect-default` -> Upload connector config file -> `connector_datagen-pageviews_config.json` -> Continue -> Launch (Should see : Running)
  Cluster 1 -> Topics -> `pageviews` -> Messages (messages should appears)

# Check if messages arrives in Snowflake table
  ```SELECT * FROM KAFKA_DB.PUBLIC.PAGEVIEWS;
  SELECT * FROM TABLE(information_schema.copy_history(table_name=>'KAFKA.PUBLIC.PAGEVIEWS', start_time=> dateadd(hours, -1, current_timestamp())));```

Mike Uzan - Senior SE (EMEA/France)
mike.uzan@snowflake.com
+33621728792
