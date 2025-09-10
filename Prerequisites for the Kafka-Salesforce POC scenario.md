Of course. Here is a step-by-step "how-to" guide for installing and configuring the prerequisites for the Kafka-Salesforce POC scenario.

### Licensing: Free Open Source for the POC ✅

For this proof-of-concept, we will exclusively use **free and open-source software**. The components we'll install are:

  * **Docker and Docker Compose:** To easily manage and run all our services in isolated containers.
  * **PostgreSQL:** Our source relational database.
  * **Apache Kafka & Zookeeper:** The core messaging platform.
  * **Kafka Connect:** A framework for connecting Kafka with external systems.
  * **Debezium:** The Change Data Capture (CDC) platform, which runs as a Kafka Connect connector.

While enterprise-grade, paid versions of these tools exist (like Confluent Platform for Kafka or Salesforce's paid subscriptions), they are not necessary for this POC. The only "paid" component is the Salesforce Developer Edition org, which is **free** for development and testing purposes.

-----

### Prerequisites Installation Guide 🛠️

This guide uses **Docker and Docker Compose** to simplify the setup process significantly. It allows us to define and run all the required services with a single command.

#### Step 1: Install Docker

First, ensure you have Docker and Docker Compose installed on your machine. They are essential for running the containerized environment.

1.  **Install Docker Desktop:** Download and install it from the official Docker website. It includes Docker Engine, Docker CLI, and Docker Compose.

      * [Docker for Mac](https://www.google.com/search?q=https.docker.com/products/docker-desktop/)
      * [Docker for Windows](https://www.google.com/search?q=https.docker.com/products/docker-desktop/)
      * [Docker for Linux](https://docs.docker.com/engine/install/)

2.  **Verify Installation:** Open your terminal or command prompt and run the following commands to ensure Docker is running correctly.

    ```bash
    docker --version
    docker-compose --version
    ```

    You should see the installed versions printed to the console without any errors.

-----

#### Step 2: Set Up the Project Directory

Create a new folder for your project. Inside this folder, you will create a `docker-compose.yml` file. This file is the blueprint for our entire backend infrastructure.

1.  Create a project directory:

    ```bash
    mkdir kafka-salesforce-poc
    cd kafka-salesforce-poc
    ```

2.  Create the `docker-compose.yml` file within this directory.

    ```bash
    touch docker-compose.yml
    ```

-----

#### Step 3: Define the Services in `docker-compose.yml`

Open the `docker-compose.yml` file in your favorite text editor and paste the following configuration. This configuration defines four essential services: Zookeeper, Kafka, PostgreSQL, and Kafka Connect (with the Debezium connector).

```yaml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.3.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.3.0
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0

  postgres:
    image: debezium/postgres:14
    container_name: postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: sourcedb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    # This command is crucial for enabling logical replication for CDC
    command: -c 'wal_level=logical'

  connect:
    image: debezium/connect:2.1
    container_name: connect
    ports:
      - "8083:8083"
    depends_on:
      - kafka
      - postgres
    environment:
      BOOTSTRAP_SERVERS: 'kafka:29092'
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: my_connect_configs
      OFFSET_STORAGE_TOPIC: my_connect_offsets
      STATUS_STORAGE_TOPIC: my_connect_statuses
```

**What this file does:**

  * **zookeeper:** Manages the Kafka cluster's state. Kafka depends on it.
  * **kafka:** The message broker that will receive change events. It's configured to be accessible from both inside (`kafka:29092`) and outside (`localhost:9092`) the Docker network.
  * **postgres:** A PostgreSQL database image specially configured for Debezium. It sets `wal_level=logical`, which is **mandatory** for Change Data Capture to work.
  * **connect:** The Kafka Connect service running the Debezium connector. It's configured to communicate with the Kafka and PostgreSQL containers.

-----

#### Step 4: Launch the Environment

Now, you can start all the services with a single command from your project directory.

1.  Run the following command in your terminal:

    ```bash
    docker-compose up -d
    ```

    The `-d` flag runs the containers in detached mode (in the background). Docker will now download the images and start the containers. This might take a few minutes the first time.

2.  **Verify that all containers are running:**

    ```bash
    docker ps
    ```

    You should see four running containers: `zookeeper`, `kafka`, `postgres`, and `connect`.

-----

#### Step 5: Set Up the PostgreSQL Database and Table

Next, we need to connect to the PostgreSQL database to create a sample table that Debezium will monitor.

1.  **Connect to the PostgreSQL container:**

    ```bash
    docker exec -it postgres psql -U user -d sourcedb
    ```

2.  **Create a sample table.** Inside the `psql` shell, run the following SQL command. This table will represent the data in our remote application.

    ```sql
    CREATE TABLE customers (
        id SERIAL PRIMARY KEY,
        first_name VARCHAR(255) NOT NULL,
        last_name VARCHAR(255) NOT NULL,
        email VARCHAR(255) UNIQUE NOT NULL,
        status VARCHAR(50)
    );
    ```

3.  **Exit the psql shell** by typing `\q` and pressing Enter.

-----

#### Step 6: Configure and Register the Debezium Connector

The final step is to tell Kafka Connect to start monitoring our `customers` table using the Debezium PostgreSQL connector. We do this by sending a JSON configuration to the Kafka Connect REST API.

1.  **Create the connector configuration.** You will send a `POST` request to the connect service's REST API on port `8083`. You can use a tool like `curl` or Postman.

    Here is the `curl` command. Run this from your terminal:

    ```bash
    curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ \
    -d '{
      "name": "salesforce-pg-connector",
      "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "database.hostname": "postgres",
        "database.port": "5432",
        "database.user": "user",
        "database.password": "password",
        "database.dbname": "sourcedb",
        "database.server.name": "pg-server-1",
        "table.include.list": "public.customers",
        "plugin.name": "pgoutput"
      }
    }'
    ```

    **Understanding the configuration:**

      * `name`: A unique name for our connector instance.
      * `connector.class`: Specifies the Debezium PostgreSQL connector.
      * `database.*`: Connection details for our PostgreSQL container. We use the service name `postgres` as the hostname because we are within the Docker network.
      * `database.server.name`: A logical name for the source database server. Debezium uses this to create a unique Kafka topic name, which will be `pg-server-1.public.customers`.
      * `table.include.list`: A comma-separated list of tables to monitor. We are only watching `public.customers`.

2.  **Verify the connector is running.** You should get an `HTTP/1.1 201 Created` response. You can double-check the status with this command:

    ```bash
    curl -H "Accept:application/json" localhost:8083/connectors/salesforce-pg-connector/status
    ```

    The response should show the connector's state as `"RUNNING"`.

-----

### Verification: Watch the Magic Happen\! ✨

Your entire prerequisite environment is now running. Debezium is monitoring the `customers` table, and any change will be sent to a Kafka topic.

Let's test it:

1.  **Start a Kafka consumer:** Open a new terminal and run this command to listen to the Kafka topic.

    ```bash
    docker exec -it kafka kafka-console-consumer \
    --bootstrap-server localhost:9092 \
    --topic pg-server-1.public.customers \
    --from-beginning \
    --property print.key=true
    ```

    This command will hang, waiting for messages.

2.  **Make a change in PostgreSQL:** Open another terminal and insert a record into the `customers` table.

    ```bash
    docker exec -it postgres psql -U user -d sourcedb -c \
    "INSERT INTO customers (first_name, last_name, email, status) VALUES ('samantha', 'jones', 's.jones@example.com', 'Active');"
    ```

3.  **Check the consumer terminal.** Almost instantly, you will see a complex JSON message appear. This is the Change Data Capture event from Debezium\! It contains the new row's data in the `after` field.

You have successfully set up all the necessary prerequisites. Your system is now capturing database changes in real-time and publishing them to Kafka, ready to be integrated with Salesforce.
