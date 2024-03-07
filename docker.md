# Docker

## Dockerfile

A document that contains all the commands a user could call on the command line to assemble an image.

#### Example for React App

```docker
# Start your image with a node base image
FROM node:18-alpine

# The /app directory should act as the main application directory
WORKDIR /app

# Copy the app package and package-lock.json file
COPY package*.json ./

# Copy local directories to the current local directory of our docker image (/app)
COPY ./src ./src
COPY ./public ./public

# Install node packages, install serve, build the app, and remove dependencies at the end
RUN npm install \
    && npm install -g serve \
    && npm run build \
    && rm -fr node_modules

EXPOSE 3000

# Start the app using serve command
CMD [ "serve", "-s", "build" ]
```

#### Example for Java Data API

```docker
FROM maven:3.8.5-openjdk-17 AS build
COPY . .
RUN mvn clean package -DskipTests

FROM openjdk:17.0.1-jdk-slim
COPY --from=build /target/annas_restaurant_recommender-0.0.1-SNAPSHOT.jar annas_restaurant_recommender.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","annas_restaurant_recommender.jar"]
```

## Docker Build vs Docker Compose Up

#### Build

```bash
docker --context desktop-linux build -t welcome-to-docker .
```

This command is used to create a Docker image from a Dockerfile. Typically used before pushing the image to a Docker registry such as Docker Hub before running a container based on this image.

The `--context` tag is used for different instances of docker and `-t` is used to name the image while `.` is the location where the Dockerfile can be found.

#### Compose Up

```bash
docker-compose up -d
```

The `docker-compose up` command is instead used to start and run the entire Docker application that is defined in a `docker-compose.yml` file. This is for multi-container Docker applications and with this command you can create and start all the containers defined in the `yml` file. 

E.g. your application might require a web server container, database container, cache container etc.

In the `yml` file you can specify specific images from Docker Hub instead of multiple Dockerfiles. At the same time, it is used for orchestrating the containers in that specific application.

## Example YAML

```yml
---
version: '2'
services:

  connect-1: #one of the two kafka connect worker nodes 
    image: confluentinc/cp-server-connect:7.1.1
    restart: always
    hostname: connect-1
    container_name: connect-1
    ports:
      - "8083:8083"
    volumes:
      - ./data:/data
    environment:
      CONNECT_BOOTSTRAP_SERVERS: $BOOTSTRAP_SERVERS #environment variable, endpoint for kc101 cluster
      CONNECT_GROUP_ID: "kc101-connect" #tells what connect cluster to join
      #internal topics to keep in sync with one another
      CONNECT_CONFIG_STORAGE_TOPIC: "_kc101-connect-configs" 
      CONNECT_OFFSET_STORAGE_TOPIC: "_kc101-connect-offsets"
      CONNECT_STATUS_STORAGE_TOPIC: "_kc101-connect-status"
      CONNECT_REPLICATION_FACTOR: 3
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 3
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 3
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 3
      CONNECT_KEY_CONVERTER: "org.apache.kafka.connect.storage.StringConverter"

      # Confluent Schema Registry for Kafka Connect
      CONNECT_VALUE_CONVERTER: "io.confluent.connect.avro.AvroConverter"
      CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE: "true"
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: $SCHEMA_REGISTRY_URL
      CONNECT_VALUE_CONVERTER_BASIC_AUTH_CREDENTIALS_SOURCE: $BASIC_AUTH_CREDENTIALS_SOURCE
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_BASIC_AUTH_USER_INFO: $SCHEMA_REGISTRY_BASIC_AUTH_USER_INFO

      CONNECT_REST_ADVERTISED_HOST_NAME: "connect-1"
      CONNECT_LISTENERS: http://connect-1:8083
      CONNECT_PLUGIN_PATH: "/usr/share/java,/usr/share/confluent-hub-components"
      CONNECT_LOG4J_APPENDER_STDOUT_LAYOUT_CONVERSIONPATTERN: "[%d] %p %X{connector.context}%m (%c:%L)%n"
      CONNECT_LOG4J_ROOT_LOGLEVEL: INFO
      CONNECT_LOG4J_LOGGERS: 'org.apache.kafka.connect.runtime.rest=WARN,org.reflections=ERROR'

      # Confluent Cloud config
      CONNECT_REQUEST_TIMEOUT_MS: "20000"
      CONNECT_RETRY_BACKOFF_MS: "500"
      CONNECT_SSL_ENDPOINT_IDENTIFICATION_ALGORITHM: "https"

      # Connect worker
      CONNECT_SECURITY_PROTOCOL: SASL_SSL
      CONNECT_SASL_JAAS_CONFIG: $SASL_JAAS_CONFIG
      CONNECT_SASL_MECHANISM: PLAIN

      # Connect producer
      CONNECT_PRODUCER_SECURITY_PROTOCOL: SASL_SSL
      CONNECT_PRODUCER_SASL_JAAS_CONFIG: $SASL_JAAS_CONFIG
      CONNECT_PRODUCER_SASL_MECHANISM: PLAIN

      # Connect consumer
      CONNECT_CONSUMER_SECURITY_PROTOCOL: SASL_SSL
      CONNECT_CONSUMER_SASL_JAAS_CONFIG: $SASL_JAAS_CONFIG
      CONNECT_CONSUMER_SASL_MECHANISM: PLAIN

    #installing connector plugins that will be used, downloaded from confluent hub
    command:
      - bash
      - -c
      - |
        echo "Installing Connector"
        confluent-hub install --no-prompt debezium/debezium-connector-mysql:1.7.1
        confluent-hub install --no-prompt confluentinc/kafka-connect-elasticsearch:11.1.8
        confluent-hub install --no-prompt neo4j/kafka-connect-neo4j:2.0.1
        confluent-hub install --no-prompt confluentinc/kafka-connect-jdbc:10.3.3
        #
        echo "Launching Kafka Connect worker"
        /etc/confluent/docker/run &
        #
        sleep infinity

  connect-2: #second node
    image: confluentinc/cp-server-connect:7.1.1
    restart: always
    hostname: connect-2
    container_name: connect-2
    ports:
      - "8084:8084"
    volumes:
      - ./data:/data
    environment:
      CONNECT_BOOTSTRAP_SERVERS: $BOOTSTRAP_SERVERS
      CONNECT_GROUP_ID: "kc101-connect"
      #internal topics the worker nodes use to keep in sync with one another
      CONNECT_CONFIG_STORAGE_TOPIC: "_kc101-connect-configs" 
      CONNECT_OFFSET_STORAGE_TOPIC: "_kc101-connect-offsets"
      CONNECT_STATUS_STORAGE_TOPIC: "_kc101-connect-status"
      CONNECT_REPLICATION_FACTOR: 3
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 3
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 3
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 3
      CONNECT_KEY_CONVERTER: "org.apache.kafka.connect.storage.StringConverter"

      # Confluent Schema Registry for Kafka Connect
      CONNECT_VALUE_CONVERTER: "io.confluent.connect.avro.AvroConverter"
      CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE: "true"
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: $SCHEMA_REGISTRY_URL
      CONNECT_VALUE_CONVERTER_BASIC_AUTH_CREDENTIALS_SOURCE: $BASIC_AUTH_CREDENTIALS_SOURCE
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_BASIC_AUTH_USER_INFO: $SCHEMA_REGISTRY_BASIC_AUTH_USER_INFO

      CONNECT_REST_ADVERTISED_HOST_NAME: "connect-2"
      CONNECT_LISTENERS: http://connect-2:8084
      CONNECT_PLUGIN_PATH: "/usr/share/java,/usr/share/confluent-hub-components"
      CONNECT_LOG4J_APPENDER_STDOUT_LAYOUT_CONVERSIONPATTERN: "[%d] %p %X{connector.context}%m (%c:%L)%n"
      CONNECT_LOG4J_ROOT_LOGLEVEL: INFO
      CONNECT_LOG4J_LOGGERS: 'org.apache.kafka.connect.runtime.rest=WARN,org.reflections=ERROR'

      # Confluent Cloud config
      CONNECT_REQUEST_TIMEOUT_MS: "20000"
      CONNECT_RETRY_BACKOFF_MS: "500"
      CONNECT_SSL_ENDPOINT_IDENTIFICATION_ALGORITHM: "https"

      # Connect worker
      CONNECT_SECURITY_PROTOCOL: SASL_SSL
      CONNECT_SASL_JAAS_CONFIG: $SASL_JAAS_CONFIG
      CONNECT_SASL_MECHANISM: PLAIN

      # Connect producer
      CONNECT_PRODUCER_SECURITY_PROTOCOL: SASL_SSL
      CONNECT_PRODUCER_SASL_JAAS_CONFIG: $SASL_JAAS_CONFIG
      CONNECT_PRODUCER_SASL_MECHANISM: PLAIN

      # Connect consumer
      CONNECT_CONSUMER_SECURITY_PROTOCOL: SASL_SSL
      CONNECT_CONSUMER_SASL_JAAS_CONFIG: $SASL_JAAS_CONFIG
      CONNECT_CONSUMER_SASL_MECHANISM: PLAIN

    command:
      - bash
      - -c
      - |
        echo "Installing Connector"
        confluent-hub install --no-prompt debezium/debezium-connector-mysql:1.7.1
        confluent-hub install --no-prompt confluentinc/kafka-connect-elasticsearch:11.1.8
        confluent-hub install --no-prompt neo4j/kafka-connect-neo4j:2.0.1
        confluent-hub install --no-prompt confluentinc/kafka-connect-jdbc:10.3.3
        #
        echo "Launching Kafka Connect worker"
        /etc/confluent/docker/run &
        #
        sleep infinity

# Other systems
  mysql:
    # *-----------------------------*
    # To connect to the DB:
    #   docker exec -it mysql bash -c 'mysql -u root -p$MYSQL_ROOT_PASSWORD demo'
    # *-----------------------------*
    image: mysql:8.0
    container_name: mysql
    ports:
      - 3306:3306
    environment:
     - MYSQL_ROOT_PASSWORD=kc101
     - MYSQL_USER=mysqluser
     - MYSQL_PASSWORD=mysqlpw
    volumes:
     - ${PWD}/data/mysql:/docker-entrypoint-initdb.d
     - ${PWD}/data:/data

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
    container_name: elasticsearch
    hostname: elasticsearch
    ports:
      - 9200:9200
    environment:
      xpack.security.enabled: "false"
      ES_JAVA_OPTS: "-Xms1g -Xmx1g"
      discovery.type: "single-node"

  neo4j:
    image: neo4j:4.4.4
    container_name: neo4j
    ports:
    - "7474:7474"
    - "7687:7687"
    environment:
      NEO4J_AUTH: neo4j/connect
      NEO4J_dbms_memory_heap_max__size: 2G
      NEO4J_dbms_memory_pagecache_size: 1G
      NEO4J_ACCEPT_LICENSE_AGREEMENT: 'yes'
```

