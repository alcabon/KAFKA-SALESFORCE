# **A Proof-of-Concept Guide: Real-Time PostgreSQL to Salesforce Integration via Debezium, Kafka, and Platform Events**

## **Architectural Blueprint and Core Concepts**

### **The Event-Driven Paradigm in Enterprise Integration**

Modern enterprise ecosystems are characterized by a distributed network of specialized applications, each serving as a system of record or a system of engagement. A common and critical challenge is maintaining data consistency and timeliness across these disparate systems. For instance, a remote application utilizing a PostgreSQL database may act as the definitive source for customer data, while Salesforce serves as the primary platform for sales, service, and marketing engagement. Traditional integration methods, often reliant on batch processing or periodic API polling, introduce significant latency, place a heavy load on source systems, and result in brittle, tightly-coupled point-to-point connections.1

An event-driven architecture (EDA) offers a superior alternative by fundamentally inverting the control flow of data. Instead of a destination system repeatedly asking, "Has anything changed?", the source system proactively announces changes as they occur. These announcements, known as events, are broadcast to a central message bus. Downstream systems can then subscribe to these event streams and react in near real-time. This model promotes a decoupled, scalable, and highly responsive integration landscape.1 An event is a meaningful change in state, an event message contains the data about that change, and the event bus is the channel through which producers transmit these messages to consumers.1

At the heart of this modern architectural pattern is Apache Kafka, the de facto standard for high-throughput, fault-tolerant event streaming. Kafka's log-based architecture provides a durable, time-ordered record of all events, allowing multiple consumers to subscribe to the same data stream independently and at their own pace.2 The robustness and scalability of Kafka are validated by its adoption within Salesforce's own infrastructure, where it powers core features like the Streaming API and Enterprise Messaging, underscoring its suitability for enterprise-grade solutions.4 This proof-of-concept (PoC) will construct a complete event-driven pipeline leveraging these principles to synchronize data from PostgreSQL to Salesforce.

### **High-Level Architectural Diagram and Data Flow**

The integration pipeline is composed of several specialized components working in concert to capture, stream, transform, and deliver data changes. The end-to-end data flow proceeds as follows:

1. **Database Change:** A transaction (INSERT, UPDATE, or DELETE) is committed to a customers table within the PostgreSQL database. This change is immutably recorded in the database's internal Write-Ahead Log (WAL).  
2. **Change Data Capture (CDC):** The Debezium PostgreSQL Connector, running within the Kafka Connect framework, is configured to monitor the PostgreSQL instance. It reads the logical decoding output of the WAL, capturing the row-level change without directly impacting the source database's performance.  
3. **Event Publication:** Upon detecting a change, Debezium constructs a detailed JSON message, or "change event," that describes the operation. This event contains the data state before and after the change, along with rich metadata. Debezium then publishes this event to a dedicated Apache Kafka topic.  
4. **Event Transformation:** The Kafka Connect framework intercepts the message stream. A chain of Single Message Transforms (SMTs) is applied to the raw Debezium event. These transforms flatten the complex, nested JSON structure and enrich it with necessary metadata, preparing it for the destination system.  
5. **Event Sinking:** The Confluent Salesforce Sink Connector, also running in Kafka Connect, consumes the transformed message from the Kafka topic.  
6. **Salesforce Ingestion:** The Sink Connector authenticates with the Salesforce Platform and publishes a new Platform Event to the Salesforce Event Bus. The payload of this Platform Event contains the flattened data from the original database change.  
7. **Salesforce Consumption:** Within the Salesforce organization, subscribers listen for the new Platform Event. This PoC will demonstrate two parallel consumption patterns:  
   * A declarative **Platform Event-Triggered Flow** that uses decision logic to create, update, or delete a corresponding Contact record.  
   * A programmatic **Apex Trigger** that subscribes to the same event to perform equivalent data manipulation operations (DML).

This architecture ensures that each component has a single, well-defined responsibility, creating a robust and maintainable data pipeline.

### **Component Deep Dive: Roles and Responsibilities**

A nuanced understanding of each component's specific role is essential for successful implementation and troubleshooting.

* **PostgreSQL & Logical Replication:** The PostgreSQL database serves as the authoritative source system. The key technology enabling this integration is its native support for logical replication. Unlike trigger-based CDC or query-based polling, which can be invasive and resource-intensive, logical replication allows an external process to tap into the database's transaction log (WAL). The WAL is a highly optimized, sequential log of all database modifications, primarily used for crash recovery and replication. By configuring the wal\_level to logical, the database decodes these binary log entries into a comprehensible, streamable format that Debezium can consume, providing a non-invasive, low-latency source of truth for all data changes.6  
* **Debezium:** Debezium is a distributed, open-source platform for Change Data Capture (CDC). It is not merely a data scraper; it is a sophisticated Kafka Connect connector that understands the intricate transaction semantics of the source database. It guarantees that changes are delivered in the exact order they were committed, preserving data integrity.6 By reading directly from the transaction log, Debezium captures all row-level  
  INSERT, UPDATE, and DELETE operations, packaging them into a standardized, richly detailed event envelope. This envelope provides critical context, such as the type of operation and the state of the data before and after the change, which is essential for downstream processing.7  
* **Apache Kafka & Kafka Connect:** Apache Kafka functions as the central nervous system of this architecture. It is a distributed event streaming platform that provides a durable, fault-tolerant, and highly scalable message bus.2 Events published to Kafka topics are persisted and can be replayed, ensuring that no data is lost even if downstream consumers are temporarily unavailable. Kafka Connect is a framework built on top of Kafka that standardizes the process of streaming data between Kafka and other systems. It allows for the use of pre-built, battle-tested connectors for sources (like Debezium for PostgreSQL) and sinks (like the Salesforce connector), abstracting away the complexity of writing custom producer and consumer applications. Kafka Connect also hosts the Single Message Transform (SMT) framework, enabling in-flight message manipulation without requiring a separate stream processing engine.10  
* **Salesforce Platform Events:** Platform Events are Salesforce's native, scalable solution for enterprise messaging and event-driven architecture. Defined with a custom schema much like a custom object (with an \_\_e suffix), they facilitate communication between Salesforce processes and external systems via the Salesforce Event Bus.1 By using a Platform Event as the entry point for data into Salesforce, the integration decouples the act of data ingestion from the subsequent business logic. The Kafka sink connector's sole responsibility is to publish the event. Any number of independent processes within Salesforce—Flows, Apex triggers, Lightning Web Components—can then subscribe to this event to perform various actions. This decoupling is a cornerstone of a scalable and maintainable Salesforce architecture, as new automations can be added to react to database changes without ever modifying the core integration pipeline.5

## **Environment Setup and Source System Configuration**

This section provides the actionable steps to construct the foundational infrastructure for the PoC using Docker and to prepare the PostgreSQL source system for change data capture.

### **Deploying the Infrastructure with Docker Compose**

Docker Compose provides a streamlined way to define and run the multi-container application stack required for this PoC. The following docker-compose.yml file will provision all necessary services: Zookeeper (a prerequisite for this version of Kafka), a Kafka broker, a PostgreSQL database instance, and a Kafka Connect instance with the Debezium PostgreSQL connector pre-installed.13

Create a file named docker-compose.yml with the following content:

YAML

version: '3.8'  
services:  
  zookeeper:  
    image: quay.io/debezium/zookeeper:3.1  
    ports:  
      \- "2181:2181"  
  kafka:  
    image: quay.io/debezium/kafka:3.1  
    ports:  
      \- "9092:9092"  
    links:  
      \- zookeeper  
    environment:  
      ZOOKEEPER\_CONNECT: zookeeper:2181  
  postgres:  
    image: debezium/postgres:15  
    ports:  
      \- "5432:5432"  
    environment:  
      POSTGRES\_USER: postgres  
      POSTGRES\_PASSWORD: password  
      POSTGRES\_DB: inventory  
    command: postgres \-c wal\_level=logical  
  connect:  
    image: quay.io/debezium/connect:3.1  
    ports:  
      \- "8083:8083"  
    links:  
      \- kafka  
      \- postgres  
    environment:  
      BOOTSTRAP\_SERVERS: kafka:9092  
      GROUP\_ID: 1  
      CONFIG\_STORAGE\_TOPIC: my\_connect\_configs  
      OFFSET\_STORAGE\_TOPIC: my\_connect\_offsets  
      STATUS\_STORAGE\_TOPIC: my\_connect\_statuses

To launch the environment, navigate to the directory containing this file in a terminal and execute:  
docker-compose up \-d  
This command will download the necessary images and start all four services in detached mode. The key services will be accessible on the host machine at the following ports: PostgreSQL on 5432, Kafka on 9092, and the Kafka Connect REST API on 8083\.

### **Configuring PostgreSQL for Logical Replication**

For Debezium to capture changes, the PostgreSQL instance must be correctly configured to enable logical replication. The debezium/postgres Docker image used in the docker-compose.yml file already sets the most critical parameter, wal\_level=logical, via a command-line argument.13 This setting instructs PostgreSQL to write additional information to its Write-Ahead Log (WAL) that allows for the logical decoding of row-level changes.7

In a production, non-containerized environment, this would be set in the postgresql.conf file, and the database would require a restart. Additionally, two other parameters are crucial for robust operation 8:

* max\_wal\_senders: Defines the maximum number of concurrent connections from streaming replication clients. This should be set to a value greater than the number of expected CDC connectors.  
* max\_replication\_slots: Defines the maximum number of replication slots. Each Debezium connector requires its own replication slot to track its progress in the WAL. If the connector disconnects, the slot ensures that the WAL files it still needs are not purged by the database until it reconnects and processes them.

Next, a dedicated database user with the necessary permissions for replication must be created. This follows the principle of least privilege, ensuring the connector has only the access it requires. Connect to the running PostgreSQL container and create the user 7:

1. Execute a psql shell inside the container:  
   docker-compose exec postgres psql \-U postgres \-d inventory  
2. Within the psql shell, run the following SQL commands to create the replication user and grant permissions:  
   SQL  
   CREATE USER repuser WITH REPLICATION ENCRYPTED PASSWORD 'repuserpass';  
   GRANT ALL PRIVILEGES ON DATABASE inventory TO repuser;

### **Preparing the Source Table**

The final step in preparing the source system is to create the table that will be monitored for changes and to configure its replication identity.

1. Using the same psql shell from the previous step, create a customers table:  
   SQL  
   CREATE TABLE customers (  
       id SERIAL PRIMARY KEY,  
       first\_name VARCHAR(255) NOT NULL,  
       last\_name VARCHAR(255) NOT NULL,  
       email VARCHAR(255) NOT NULL UNIQUE  
   );

2. Grant the repuser permissions on this new table:  
   SQL  
   GRANT SELECT ON customers TO repuser;  
   ALTER TABLE customers OWNER TO repuser;

3. Set the table's replica identity. This is a non-negotiable prerequisite for a meaningful CDC pipeline that correctly handles UPDATE and DELETE operations. The REPLICA IDENTITY setting determines how much information is written to the WAL for these operations. The default setting is often insufficient. Setting it to FULL forces PostgreSQL to log the complete, previous state of the entire row in the WAL.6 Without this setting, the  
   before block in the Debezium change event for an update or delete would be incomplete or null, robbing the downstream integration of critical contextual information needed for auditing or complex logic. Execute the following command:  
   SQL  
   ALTER TABLE customers REPLICA IDENTITY FULL;

With the infrastructure deployed and the source database correctly configured, the system is now ready for the Debezium connector to begin capturing data changes.

## **Capturing Data Changes with the Debezium PostgreSQL Connector**

This section details the configuration and deployment of the Debezium connector, which forms the bridge between the PostgreSQL database and the Kafka event stream. It also provides a detailed analysis of the change event structure produced by the connector.

### **Debezium Connector Configuration Deep Dive**

The Debezium connector is configured and managed via the Kafka Connect REST API. A JSON configuration file defines all the necessary parameters for the connector to connect to the database, identify which tables to monitor, and specify how to format the output topics.6

Create a file named register-postgres.json with the following content. This configuration instructs the connector to monitor the customers table within the public schema and to publish change events to a Kafka topic prefixed with pg-source.13

JSON

{  
  "name": "inventory-connector",  
  "config": {  
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",  
    "database.hostname": "postgres",  
    "database.port": "5432",  
    "database.user": "repuser",  
    "database.password": "repuserpass",  
    "database.dbname": "inventory",  
    "topic.prefix": "pg-source",  
    "table.include.list": "public.customers",  
    "publication.autocreate.mode": "filtered",  
    "plugin.name": "pgoutput"  
  }  
}

A breakdown of the key configuration properties:

* connector.class: Specifies the Java class for the Debezium PostgreSQL connector.  
* database.\*: These properties provide the connection credentials for the PostgreSQL database instance running in Docker. The hostname postgres is resolvable within the Docker network.  
* topic.prefix: This string serves as a logical name for the source database and is used as a prefix for all Kafka topics created by this connector. For the public.customers table, the resulting topic will be pg-source.public.customers.  
* table.include.list: A comma-separated list of tables to capture. This scopes the connector's operation to only the desired tables, preventing unwanted data from being streamed.  
* publication.autocreate.mode: Setting this to filtered is a best practice. It instructs the connector to create a PostgreSQL publication that includes only the tables specified in table.include.list, which is more efficient than capturing all tables.8  
* plugin.name: Specifies the logical decoding output plugin to use. pgoutput is the standard logical decoding plugin in modern PostgreSQL versions.

### **Registering and Verifying the Connector**

With the configuration file created, the connector can be registered with the Kafka Connect service using a simple HTTP POST request.

1. From a terminal in the same directory as register-postgres.json, execute the following curl command 6:  
   Bash  
   curl \-i \-X POST \-H "Accept:application/json" \-H "Content-Type:application/json" http://localhost:8083/connectors/ \-d @register-postgres.json

   A successful request will return an HTTP 201 Created status code along with the configuration of the newly created connector.  
2. To verify that the connector is running correctly, query its status 6:  
   Bash  
   curl \-H "Accept:application/json" localhost:8083/connectors/inventory-connector/status | jq

   The output should show the connector state as "RUNNING". If it shows "FAILED", the Kafka Connect logs should be inspected for errors (docker-compose logs connect).  
3. The ultimate verification is to observe the messages on the Kafka topic. Start a console consumer to listen to the pg-source.public.customers topic. This tool will print any messages published to the topic to the console.6  
   Bash  
   docker-compose exec kafka /kafka/bin/kafka-console-consumer.sh \--bootstrap-server kafka:9092 \--topic pg-source.public.customers \--from-beginning

   Initially, if there is any existing data in the customers table, Debezium will perform an initial snapshot and the consumer will display a series of messages for those records. Subsequently, it will wait for new change events.

### **Anatomy of a Debezium Change Event**

To effectively transform the data for Salesforce, it is crucial to understand the structure of the JSON event that Debezium produces. The event is not just a flat representation of the database row; it is a rich, self-describing "envelope" containing extensive metadata about the change.9 This structure, while powerful, presents a transformation challenge for sinks that expect a simple data record.

Let's examine the structure for each type of DML operation.

INSERT Operation (op: "c")  
If we execute INSERT INTO customers (first\_name, last\_name, email) VALUES ('Anne', 'Jones', 'ajones@example.com'); in PostgreSQL, the Debezium event will look like this:

JSON

{  
  "schema": {... },  
  "payload": {  
    "before": null,  
    "after": {  
      "id": 101,  
      "first\_name": "Anne",  
      "last\_name": "Jones",  
      "email": "ajones@example.com"  
    },  
    "source": {... },  
    "op": "c",  
    "ts\_ms": 1672531200000,  
    "transaction": null  
  }  
}

Key fields:

* op: "c" for create (or "r" for read during a snapshot).  
* before: null, as there was no previous state for this record.  
* after: An object containing the complete state of the newly inserted row.

UPDATE Operation (op: "u")  
If we execute UPDATE customers SET email \= 'anne.jones@work.com' WHERE id \= 101;, the event will be:

JSON

{  
  "schema": {... },  
  "payload": {  
    "before": {  
      "id": 101,  
      "first\_name": "Anne",  
      "last\_name": "Jones",  
      "email": "ajones@example.com"  
    },  
    "after": {  
      "id": 101,  
      "first\_name": "Anne",  
      "last\_name": "Jones",  
      "email": "anne.jones@work.com"  
    },  
    "source": {... },  
    "op": "u",  
    "ts\_ms": 1672531260000,  
    "transaction": null  
  }  
}

Key fields:

* op: "u" for update.  
* before: The complete state of the row *before* the update was applied. This is only fully populated because REPLICA IDENTITY FULL was set.  
* after: The complete state of the row *after* the update.

DELETE Operation (op: "d")  
If we execute DELETE FROM customers WHERE id \= 101;, the event will be:

JSON

{  
  "schema": {... },  
  "payload": {  
    "before": {  
      "id": 101,  
      "first\_name": "Anne",  
      "last\_name": "Jones",  
      "email": "anne.jones@work.com"  
    },  
    "after": null,  
    "source": {... },  
    "op": "d",  
    "ts\_ms": 1672531320000,  
    "transaction": null  
  }  
}

Key fields:

* op: "d" for delete.  
* before: The state of the row just before it was deleted.  
* after: null, as the row no longer exists.

This detailed event structure provides all the necessary information for the Salesforce integration, but its nested nature requires transformation before it can be mapped to a flat Salesforce Platform Event.

## **Designing and Defining the Salesforce Platform Event**

With the data capture mechanism in place, the focus now shifts to the destination system. This section covers the design and creation of the Salesforce Platform Event that will serve as the data contract and entry point for the PostgreSQL changes into the Salesforce ecosystem.

### **Platform Event Design Principles**

The schema of the Platform Event is a critical design decision. It defines the structure of the data that will be available to all subscribers within Salesforce. A key debate in event design is whether to create highly specific events (e.g., AccountNameChangeEvent\_\_e, AccountPhoneChangeEvent\_\_e) or more generic ones (e.g., AccountChangeEvent\_\_e).16

* **Specific Events:** Offer the advantage that subscribers only receive notifications they explicitly care about, reducing processing overhead. However, this can lead to a proliferation of event types and automation rules, increasing management complexity.  
* **Generic Events:** Consolidate changes for a given object into a single event stream. This simplifies the publishing logic and reduces the number of event definitions to manage. The trade-off is that subscribers may receive events for changes they are not interested in, requiring them to filter the events within their own logic.

For this database replication use case, a hybrid approach is most effective. A single, generic event will be created to represent any change from the source system, but it will contain specific metadata fields that allow subscribers to easily understand and act upon the event.

### **Step-by-Step Guide to Creating the Platform Event**

The following steps outline the process for creating the custom Platform Event within the Salesforce Setup menu.17

1. **Navigate to Platform Events:** From **Setup**, enter Platform Events in the Quick Find box, then select **Platform Events**.  
2. **Create a New Platform Event:** On the Platform Events page, click **New Platform Event**.  
3. **Define Event Details:**  
   * **Label:** Database Change Event  
   * **Plural Label:** Database Change Events  
   * **API Name:** Database\_Change\_Event\_\_e (This will be auto-populated).  
   * **Publish Behavior:** Select **Publish After Commit**. This ensures that the event is only published after the transaction that published it has been successfully committed to the Salesforce database. This is the standard and safest option.  
4. **Save the Event:** Click **Save**.

After the event is defined, custom fields must be added to its schema. These fields will hold the data from the PostgreSQL customers table and the essential metadata from the Debezium event envelope.

5. **Add Custom Fields:** On the Database Change Event detail page, scroll to the **Custom Fields & Relationships** section and click **New**. Create the following fields:  
   * **Field 1: Source Record ID**  
     * Data Type: **Text**  
     * Field Label: Source Record ID  
     * Length: 255  
     * Field Name: Source\_Record\_ID  
     * **Required:** Check "Always require a value in this field..."  
     * **External ID:** Check "Set this field as the unique record identifier from an external system". This is a critical setting that will allow for efficient upsert operations later.  
   * **Field 2: Operation**  
     * Data Type: **Text**  
     * Field Label: Operation  
     * Length: 1  
     * Field Name: Operation  
     * Required: Check "Always require a value in this field..."  
   * **Field 3: Source Table**  
     * Data Type: **Text**  
     * Field Label: Source Table  
     * Length: 255  
     * Field Name: Source\_Table  
   * **Field 4: First Name**  
     * Data Type: **Text**  
     * Field Label: First Name  
     * Length: 255  
     * Field Name: First\_Name  
   * **Field 5: Last Name**  
     * Data Type: **Text**  
     * Field Label: Last Name  
     * Length: 255  
     * Field Name: Last\_Name  
   * **Field 6: Email**  
     * Data Type: **Text**  
     * Field Label: Email  
     * Length: 255  
     * Field Name: Email

The inclusion of the Operation\_\_c and Source\_Record\_ID\_\_c fields is what makes this Platform Event schema truly actionable. When a subscriber in Salesforce receives an instance of Database\_Change\_Event\_\_e, it will not just receive a payload of data; it will receive the necessary context to process it correctly. The subscriber's logic will first inspect the Operation\_\_c field to determine *what to do*—create, update, or delete. Then, for updates and deletes, it will use the Source\_Record\_ID\_\_c value to find the specific Salesforce record that corresponds to the source PostgreSQL record. Without these metadata fields, the event would be ambiguous and the automation would fail.

## **Bridging the Gap: Transforming and Sinking Events into Salesforce**

This section addresses the technical core of the integration pipeline: transforming the complex Debezium change event into a flat structure and sinking it into Salesforce as the custom Platform Event defined in the previous section.

### **The Transformation Challenge: From Nested Envelope to Flat Event**

As established in Section 3, there is a significant structural mismatch between the Debezium event and the target Salesforce Platform Event.

* **Debezium Event Structure:** A nested JSON object with a payload containing before and after objects, an op field for the operation type, and a source object with metadata.  
* **Salesforce Platform Event Structure:** A flat object with fields like First\_Name\_\_c, Email\_\_c, and the custom metadata fields Operation\_\_c and Source\_Record\_ID\_\_c.

A direct mapping is impossible. The pipeline requires an intermediate transformation step to "unwrap" the relevant data from the Debezium envelope and restructure it to match the target schema.

### **Solution: Kafka Connect Single Message Transforms (SMTs)**

Kafka Connect provides a powerful, built-in mechanism for performing lightweight, in-flight message transformations called Single Message Transforms (SMTs).21 SMTs are configured directly on the sink connector and operate on each message as it flows from the Kafka topic to the sink system. This declarative approach avoids the need for a separate, complex stream processing application (like Kafka Streams or Flink) for many common transformation tasks.23

For this PoC, a chain of SMTs will be configured to perform the necessary restructuring:

1. **io.debezium.transforms.ExtractNewRecordState:** This is the primary transformation provided by Debezium, often referred to as the "unwrap" transform. Its function is to extract the after block from a Debezium create or update event, or the before block from a delete event, and replace the entire complex event with this simple, flat JSON object. This effectively discards the event envelope.23  
2. **org.apache.kafka.connect.transforms.InsertField$Value:** This standard Kafka Connect SMT is used to add static or record-derived fields to the message. It will be used to re-introduce critical metadata that was discarded by the unwrap transform, such as the operation type (op) and the source table name.  
3. **org.apache.kafka.connect.transforms.HoistField$Value:** This SMT is used to extract the primary key field (e.g., id) from the record and promote it to be the message key. While not strictly required for this sink, it's a common pattern. For this PoC, it will be used to ensure the source ID is available for mapping.  
4. **org.apache.kafka.connect.transforms.ReplaceField$Value:** This SMT can be used to rename fields to match the expected API names of the Salesforce Platform Event fields.

### **Configuring the Salesforce Platform Event Sink Connector**

The Confluent Salesforce Connector is a robust, enterprise-ready connector for integrating Kafka with Salesforce.10 It can be configured as a sink to consume records from a Kafka topic and publish them as Platform Events.

Prerequisite: Salesforce Connected App Setup  
Before configuring the connector, a Connected App must be created in Salesforce to enable secure API access for the integration. This provides the necessary OAuth credentials (Consumer Key and Consumer Secret) that the connector will use to authenticate.10

1. In Salesforce Setup, navigate to **App Manager** and click **New Connected App**.  
2. Fill in the basic information (e.g., App Name: Kafka Integration).  
3. Under **API (Enable OAuth Settings)**, check **Enable OAuth Settings**.  
4. Add **Access and manage your data (api)** and **Perform requests on your behalf at any time (refresh\_token, offline\_access)** to the Selected OAuth Scopes.  
5. Save the Connected App. After saving, click **Continue**.  
6. On the app's management page, find the **Consumer Key** and **Consumer Secret**. These values, along with the Salesforce username, password, and security token of the integration user, will be needed for the connector configuration.

Sink Connector Configuration  
Create a file named register-salesforce-sink.json. This configuration defines the connection to Salesforce, specifies the target Platform Event, and, most importantly, orchestrates the chain of SMTs to perform the necessary data transformation.

JSON

{  
  "name": "salesforce-platform-event-sink",  
  "config": {  
    "connector.class": "io.confluent.connect.salesforce.SalesforcePlatformEventSinkConnector",  
    "tasks.max": "1",  
    "topics": "pg-source.public.customers",  
    "salesforce.instance": "https://login.salesforce.com",  
    "salesforce.username": "your-salesforce-user@example.com",  
    "salesforce.password": "YourPasswordAndSecurityToken",  
    "salesforce.consumer.key": "YOUR\_CONNECTED\_APP\_CONSUMER\_KEY",  
    "salesforce.consumer.secret": "YOUR\_CONNECTED\_APP\_CONSUMER\_SECRET",  
    "salesforce.platform.event.name": "Database\_Change\_Event\_\_e",  
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",  
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",  
    "value.converter.schemas.enable": "false",

    "transforms": "unwrap,insertOp,insertTable,renameId,renameFields",  
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",  
    "transforms.unwrap.drop.tombstones": "false",  
    "transforms.unwrap.delete.handling.mode": "rewrite",

    "transforms.insertOp.type": "org.apache.kafka.connect.transforms.InsertField$Value",  
    "transforms.insertOp.static.field": "Operation\_\_c",  
    "transforms.insertOp.static.value": "${topic}",  
      
    "transforms.insertTable.type": "org.apache.kafka.connect.transforms.InsertField$Value",  
    "transforms.insertTable.topic.field": "Source\_Table\_\_c",

    "transforms.renameId.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",  
    "transforms.renameId.renames": "id:Source\_Record\_ID\_\_c",

    "transforms.renameFields.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",  
    "transforms.renameFields.renames": "first\_name:First\_Name\_\_c,last\_name:Last\_Name\_\_c,email:Email\_\_c"  
  }  
}

*Note: The confluentinc/kafka-connect-salesforce plugin must be installed in the Kafka Connect environment. The Debezium Docker image used does not include it by default, so in a real scenario, a custom Docker image would be built to include this plugin.*

Register this connector using the same curl command as before:  
curl \-i \-X POST \-H "Accept:application/json" \-H "Content-Type:application/json" http://localhost:8083/connectors/ \-d @register-salesforce-sink.json

### **Data Mapping Table**

The SMT chain defined in the configuration file accomplishes a precise mapping from the source Debezium event to the target Salesforce Platform Event. The following table explicitly documents this transformation journey, serving as both a design specification and an operational guide.

| Debezium Event Source Field | SMT Applied | Transformed Field Name | Salesforce Platform Event Field | Notes |
| :---- | :---- | :---- | :---- | :---- |
| payload.after.id or payload.before.id | ExtractNewRecordState, then ReplaceField | Source\_Record\_ID\_\_c | Source\_Record\_ID\_\_c | The primary key from PostgreSQL is mapped to the External ID field. |
| payload.after.first\_name | ExtractNewRecordState, then ReplaceField | First\_Name\_\_c | First\_Name\_\_c | Direct mapping of the customer's first name. |
| payload.after.last\_name | ExtractNewRecordState, then ReplaceField | Last\_Name\_\_c | Last\_Name\_\_c | Direct mapping of the customer's last name. |
| payload.after.email | ExtractNewRecordState, then ReplaceField | Email\_\_c | Email\_\_c | Direct mapping of the customer's email. |
| payload.op | InsertField | Operation\_\_c | Operation\_\_c | The Debezium operation code ('c', 'u', 'd') is captured and inserted into the event. |
| Kafka Topic Name | InsertField | Source\_Table\_\_c | Source\_Table\_\_c | The source table information is derived from the Kafka topic name and inserted. |

This table clearly illustrates how the transformation logic bridges the structural gap between the systems. It shows that the final payload sent to Salesforce is a simple, flat JSON object whose keys perfectly match the API names of the Database\_Change\_Event\_\_e fields, ensuring a successful ingestion.

## **Processing Events within Salesforce: Completing the PoC**

The final stage of the proof-of-concept is to build subscribers within the Salesforce organization that listen for the Database\_Change\_Event\_\_e and translate these events into tangible data changes. This demonstrates the completion of the end-to-end data flow. A key advantage of the event-driven model is its ability to support multiple, independent consumers of the same event stream. This PoC will implement both a declarative consumer using Salesforce Flow and a programmatic consumer using an Apex Trigger, showcasing the platform's flexibility.

### **Declarative Consumption with a Platform Event-Triggered Flow**

Salesforce Flow provides a powerful, low-code tool for administrators and declarative developers to build sophisticated automation. A Platform Event-Triggered Flow automatically launches when a specific platform event message is received, making it an ideal tool for this use case.18

The following steps outline the creation of a flow to process the incoming database changes 1:

1. **Create a New Flow:** In Salesforce Setup, navigate to **Flows** and click **New Flow**. Select the **Platform Event–Triggered Flow** template.  
2. **Configure the Start Element:**  
   * Click on the **Start** element.  
   * In the **Platform Event** field, search for and select Database Change Event.  
   * The flow will now be configured to trigger every time a Database\_Change\_Event\_\_e is published to the event bus. The payload of the event message will be available throughout the flow via the $Record global variable.28  
3. **Add a Decision Element:** Drag a **Decision** element onto the canvas to route the flow based on the operation type.  
   * Label: Check Operation Type  
   * Create three outcomes:  
     * **Outcome 1 (Create):** Label Is Create. Condition: $Record.Operation\_\_c **Equals** c.  
     * **Outcome 2 (Update):** Label Is Update. Condition: $Record.Operation\_\_c **Equals** u.  
     * **Default Outcome (Delete):** Label Is Delete. This will catch the 'd' operation.  
4. **Implement the "Create" Path:**  
   * Under the Is Create path, add a **Create Records** element.  
   * Label: Create Contact  
   * How Many Records to Create: **One**  
   * How to Set the Record Fields: **Use separate resources, and literal values**  
   * Object: **Contact**  
   * Field Mappings:  
     * FirstName \= $Record.First\_Name\_\_c  
     * LastName \= $Record.Last\_Name\_\_c  
     * Email \= $Record.Email\_\_c  
     * Postgres\_ID\_\_c \= $Record.Source\_Record\_ID\_\_c (Assuming a custom text field Postgres\_ID\_\_c with External ID attribute exists on the Contact object to store the source primary key).  
5. **Implement the "Update" Path:**  
   * Under the Is Update path, first add a **Get Records** element to find the existing contact.  
     * Label: Find Contact to Update  
     * Object: **Contact**  
     * Condition: Postgres\_ID\_\_c **Equals** $Record.Source\_Record\_ID\_\_c  
   * Next, add an **Update Records** element.  
     * Label: Update Contact  
     * How to Find Records to Update: **Use the IDs and all field values from a record or record collection**.  
     * Record or Record Collection: Select the record variable created by the Find Contact to Update element.  
6. **Implement the "Delete" Path:**  
   * Under the Is Delete path, add a **Get Records** element similar to the update path to find the contact.  
     * Label: Find Contact to Delete  
     * Object: **Contact**  
     * Condition: Postgres\_ID\_\_c **Equals** $Record.Source\_Record\_ID\_\_c  
   * Next, add a **Delete Records** element.  
     * Label: Delete Contact  
     * How to Find Records to Delete: **Use the IDs from a record or record collection**.  
     * Record or Record Collection: Select the record variable from the Find Contact to Delete element.  
7. **Save and Activate:** Save the flow with a descriptive name (e.g., "Process Database Change Events") and activate it.

### **Programmatic Consumption with an Apex Trigger**

For more complex logic, custom error handling, or integration with other systems, an Apex trigger provides a programmatic solution. Apex triggers can subscribe to Platform Events by using an after insert trigger on the event's API name.1 The trigger fires after the event message is published to the event bus.

The following Apex code provides an efficient, bulkified way to process the same events 29:

Apex

trigger DatabaseChangeEventTrigger on Database\_Change\_Event\_\_e (after insert) {

    Map\<String, Database\_Change\_Event\_\_e\> createEventsMap \= new Map\<String, Database\_Change\_Event\_\_e\>();  
    Map\<String, Database\_Change\_Event\_\_e\> updateEventsMap \= new Map\<String, Database\_Change\_Event\_\_e\>();  
    List\<String\> deleteRecordIds \= new List\<String\>();

    // 1\. Segregate events by operation type  
    for (Database\_Change\_Event\_\_e event : Trigger.New) {  
        if (event.Operation\_\_c \== 'c') {  
            createEventsMap.put(event.Source\_Record\_ID\_\_c, event);  
        } else if (event.Operation\_\_c \== 'u') {  
            updateEventsMap.put(event.Source\_Record\_ID\_\_c, event);  
        } else if (event.Operation\_\_c \== 'd') {  
            deleteRecordIds.add(event.Source\_Record\_ID\_\_c);  
        }  
    }

    // 2\. Process Create and Update operations using an upsert for efficiency  
    List\<Contact\> contactsToUpsert \= new List\<Contact\>();  
      
    // Handle Creates  
    for(Database\_Change\_Event\_\_e event : createEventsMap.values()){  
        contactsToUpsert.add(new Contact(  
            FirstName \= event.First\_Name\_\_c,  
            LastName \= event.Last\_Name\_\_c,  
            Email \= event.Email\_\_c,  
            Postgres\_ID\_\_c \= event.Source\_Record\_ID\_\_c  
        ));  
    }

    // Handle Updates  
    for(Database\_Change\_Event\_\_e event : updateEventsMap.values()){  
        contactsToUpsert.add(new Contact(  
            FirstName \= event.First\_Name\_\_c,  
            LastName \= event.Last\_Name\_\_c,  
            Email \= event.Email\_\_c,  
            Postgres\_ID\_\_c \= event.Source\_Record\_ID\_\_c  
        ));  
    }

    if (\!contactsToUpsert.isEmpty()) {  
        // Use the external ID field for the upsert operation  
        upsert contactsToUpsert Contact.Fields.Postgres\_ID\_\_c;  
    }

    // 3\. Process Delete operations  
    if (\!deleteRecordIds.isEmpty()) {  
        List\<Contact\> contactsToDelete \=;  
        if (\!contactsToDelete.isEmpty()) {  
            delete contactsToDelete;  
        }  
    }  
}

This single Apex trigger can run in parallel with the Flow. Both will receive and process every Database\_Change\_Event\_\_e, demonstrating the one-to-many distribution power of the publish-subscribe model.

### **Verification**

To validate the entire pipeline from end to end, perform the following actions and observe the results in both the Kafka consumer console and the Salesforce org.

1. **INSERT Operation:**  
   * Action: In a psql client connected to the PostgreSQL container, execute:  
     INSERT INTO customers (id, first\_name, last\_name, email) VALUES (1, 'John', 'Doe', 'john.doe@example.com');  
   * **Expected Result:** A new Contact record for John Doe is created in Salesforce. The Postgres\_ID\_\_c field on this contact is populated with 1\.  
2. **UPDATE Operation:**  
   * Action: Execute:  
     UPDATE customers SET email \= 'j.doe@newcorp.com' WHERE id \= 1;  
   * **Expected Result:** The email address on the existing John Doe Contact record in Salesforce is updated to j.doe@newcorp.com.  
3. **DELETE Operation:**  
   * Action: Execute:  
     DELETE FROM customers WHERE id \= 1;  
   * **Expected Result:** The John Doe Contact record is deleted from Salesforce.

Successful completion of these verification steps confirms that the real-time data synchronization pipeline is fully functional.

## **Production-Readiness and Advanced Considerations**

While the proof-of-concept demonstrates the core functionality of the integration pattern, transitioning to a production environment requires addressing critical aspects of resilience, data consistency, monitoring, and security.

### **Error Handling and Resilience**

Distributed systems must be designed to handle failures gracefully. Both Kafka Connect and Salesforce offer mechanisms to build a resilient pipeline.

* **Kafka Connect Dead Letter Queue (DLQ):** A single malformed or unprocessable message—perhaps due to an unexpected data type or a temporary sink system outage—should not halt the entire data stream. Kafka Connect can be configured with a Dead Letter Queue (DLQ). If the sink connector fails to process a message after a configured number of retries, it writes the problematic message to a separate Kafka topic (the DLQ) along with metadata about the failure. This isolates the "poison pill" message for later analysis and allows the connector to continue processing subsequent valid messages.  
* **Salesforce Subscriber Error Handling:**  
  * **Flows:** Salesforce Flow includes Fault Paths. If an element like **Create Records** or **Update Records** fails (e.g., due to a validation rule), the flow can be routed down a fault path to perform corrective actions, such as logging the error to a custom object or sending a notification to an administrator.  
  * **Apex:** Apex triggers should be wrapped in try-catch blocks to handle DML exceptions or other runtime errors. Failed event data can be logged to a custom Integration\_Error\_\_c object for review. For transient errors, such as a temporary inability to acquire a record lock, Apex triggers can throw an EventBus.RetryableException to signal to the platform that the event processing should be retried.29

### **Data Consistency and Ordering Guarantees**

Maintaining the correct sequence of operations is paramount for data consistency. If an UPDATE event is processed before the initial INSERT event for the same record, data corruption will occur.

* **Kafka's Ordering Guarantee:** Apache Kafka guarantees the strict ordering of messages *within a single topic partition*.2 By default, Debezium publishes all changes for a single database table to a single Kafka topic. If this topic is configured with only one partition, all events (  
  INSERT, UPDATE, DELETE) for all records in that table will be stored and delivered to the consumer in the exact chronological order they were committed to the database.  
* **The Scalability Trade-off:** Using a single partition provides the strongest and simplest ordering guarantee. However, it limits the consumption throughput to a single consumer task. For extremely high-volume tables, this can become a bottleneck. The alternative is to use multiple partitions for the topic, typically partitioning messages by their primary key. This allows for parallel consumption, increasing throughput. However, it only guarantees order for events related to the *same primary key*. The consuming application must be designed to handle the possibility that events for different records may be processed out of their original commit order. For most Salesforce integration scenarios, the simplicity and strong consistency of a single-partition topic are preferred.

### **Monitoring and Alerting**

Proactive monitoring is essential for maintaining the health of a production pipeline.

* **Kafka and Debezium Monitoring:** The Debezium connectors expose a wealth of performance metrics via Java Management Extensions (JMX). These metrics should be scraped by a monitoring tool (e.g., Prometheus) and visualized in a dashboard (e.g., Grafana). The single most important metric to monitor is **connector lag**. This indicates how far behind the real-time database changes the connector is. A consistently growing lag is a clear sign of a performance bottleneck that needs investigation.8 Other key metrics include throughput (records/sec) and the number of failed connector tasks.  
* **Salesforce Monitoring:** Salesforce provides tools to monitor Platform Event usage. The PlatformEventUsageMetric object can be queried via SOQL to get hourly and daily data on the number of events published and delivered.32 This data is also available in the Salesforce Setup UI under "Platform Event Usage." Alerts should be configured to trigger if event delivery usage approaches the org's 24-hour allocation limits, as exceeding these limits will cause event delivery to be suspended.

### **Security and Credentials Management**

Hardcoding credentials in configuration files is a significant security risk and is unacceptable in a production environment.

* **Secrets Management:** Kafka Connect supports ConfigProviders, a pluggable mechanism for externalizing sensitive information like passwords and API keys. Instead of placing credentials directly in the connector's JSON configuration, a placeholder variable is used. At runtime, the ConfigProvider fetches the secret from a secure store like HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault. This practice ensures that secrets are not checked into version control and can be managed and rotated centrally.  
* **Salesforce Security:** The integration user account in Salesforce whose credentials are used by the sink connector should adhere to the principle of least privilege. Its profile should only be granted the specific object and field permissions required to create, read, update, and delete the target records (e.g., Contact). It should not be a System Administrator profile. The Connected App's OAuth scopes should also be limited to the minimum required, which is typically api and refresh\_token.

#### **Sources des citations**

1. Platform Events in Salesforce \- Apex Hours, consulté le septembre 10, 2025, [https://www.apexhours.com/platform-events-in-salesforce/](https://www.apexhours.com/platform-events-in-salesforce/)  
2. Installing Debezium, consulté le septembre 10, 2025, [https://debezium.io/documentation/reference/stable/install.html](https://debezium.io/documentation/reference/stable/install.html)  
3. Debezium, consulté le septembre 10, 2025, [https://debezium.io/](https://debezium.io/)  
4. Expanding Visibility With Apache Kafka \- Salesforce Engineering Blog, consulté le septembre 10, 2025, [https://engineering.salesforce.com/expanding-visibility-with-apache-kafka-e305b12c4aba/](https://engineering.salesforce.com/expanding-visibility-with-apache-kafka-e305b12c4aba/)  
5. How Apache Kafka Inspired Our Platform Events Architecture \- Salesforce Engineering Blog, consulté le septembre 10, 2025, [https://engineering.salesforce.com/how-apache-kafka-inspired-our-platform-events-architecture-2f351fe4cf63/](https://engineering.salesforce.com/how-apache-kafka-inspired-our-platform-events-architecture-2f351fe4cf63/)  
6. From Postgres to Kafka through Debezium, consulté le septembre 10, 2025, [https://dzlab.github.io/debezium/2024/06/09/debezium-kafka/](https://dzlab.github.io/debezium/2024/06/09/debezium-kafka/)  
7. Setting Up the Debezium PostgreSQL Connector Simplified \- Hevo Data, consulté le septembre 10, 2025, [https://hevodata.com/learn/debezium-postgresql/](https://hevodata.com/learn/debezium-postgresql/)  
8. Enabling CDC with the Fully Managed Debezium PostgreSQL ..., consulté le septembre 10, 2025, [https://www.confluent.io/blog/cdc-and-data-streaming-capture-database-changes-in-real-time-with-debezium/](https://www.confluent.io/blog/cdc-and-data-streaming-capture-database-changes-in-real-time-with-debezium/)  
9. Event Record Changes :: Debezium Documentation, consulté le septembre 10, 2025, [https://debezium.io/documentation/reference/stable/transformations/event-changes.html](https://debezium.io/documentation/reference/stable/transformations/event-changes.html)  
10. How to Load & Connect Kafka to Salesforce: Steps Explained, consulté le septembre 10, 2025, [https://hevodata.com/learn/connect-kafka-to-salesforce/](https://hevodata.com/learn/connect-kafka-to-salesforce/)  
11. How to send data from Apache Kafka to Salesforce \- RudderStack, consulté le septembre 10, 2025, [https://www.rudderstack.com/guides/send-data-from-apache-kafka-to-salesforce/](https://www.rudderstack.com/guides/send-data-from-apache-kafka-to-salesforce/)  
12. Platform Events Basics \- Trailhead \- Salesforce, consulté le septembre 10, 2025, [https://trailhead.salesforce.com/content/learn/modules/platform\_events\_basics](https://trailhead.salesforce.com/content/learn/modules/platform_events_basics)  
13. Postgres CDC with Debezium: Complete tutorial \- Sequin Blog, consulté le septembre 10, 2025, [https://blog.sequinstream.com/postgres-cdc-with-debezium-complete-step-by-step-tutorial/](https://blog.sequinstream.com/postgres-cdc-with-debezium-complete-step-by-step-tutorial/)  
14. Quick Guide: Setting up Postgres CDC with Debezium : r/dataengineering \- Reddit, consulté le septembre 10, 2025, [https://www.reddit.com/r/dataengineering/comments/1kft7bp/quick\_guide\_setting\_up\_postgres\_cdc\_with\_debezium/](https://www.reddit.com/r/dataengineering/comments/1kft7bp/quick_guide_setting_up_postgres_cdc_with_debezium/)  
15. Debezium Json | Ververica documentation, consulté le septembre 10, 2025, [https://docs.ververica.com/byoc/reference/formats/debezium-jason/](https://docs.ververica.com/byoc/reference/formats/debezium-jason/)  
16. Platform Event & Outbound Messaging Architecture Recommendations : r/salesforce, consulté le septembre 10, 2025, [https://www.reddit.com/r/salesforce/comments/1ityy3i/platform\_event\_outbound\_messaging\_architecture/](https://www.reddit.com/r/salesforce/comments/1ityy3i/platform_event_outbound_messaging_architecture/)  
17. trailhead.salesforce.com, consulté le septembre 10, 2025, [https://trailhead.salesforce.com/content/learn/modules/platform\_events\_basics/platform\_events\_define\_publish\#:\~:text=From%20Setup%2C%20enter%20Platform%20Events,Plural%20Label%2C%20enter%20Cloud%20News%20.](https://trailhead.salesforce.com/content/learn/modules/platform_events_basics/platform_events_define_publish#:~:text=From%20Setup%2C%20enter%20Platform%20Events,Plural%20Label%2C%20enter%20Cloud%20News%20.)  
18. Ultimate Guide to Using Platform Events in Salesforce Flow for Real-Time Updates, consulté le septembre 10, 2025, [https://hicglobalsolutions.com/blog/platform-events-salesforce-flow-real-time-updates/](https://hicglobalsolutions.com/blog/platform-events-salesforce-flow-real-time-updates/)  
19. Create Contact Record in Salesforce Org using Platform Event ..., consulté le septembre 10, 2025, [https://hicglobalsolutions.com/blog/create-contact-record-in-salesforce-org-using-platform-event-trigger-flow/](https://hicglobalsolutions.com/blog/create-contact-record-in-salesforce-org-using-platform-event-trigger-flow/)  
20. Article: Create Platform Events in Salesforce using Flow builder \- Boomi Community, consulté le septembre 10, 2025, [https://community.boomi.com/s/article/Create-Platform-Events-in-Salesforce-using-Flow-builder](https://community.boomi.com/s/article/Create-Platform-Events-in-Salesforce-using-Flow-builder)  
21. Transformations :: Debezium Documentation, consulté le septembre 10, 2025, [https://debezium.io/documentation/reference/stable/transformations/index.html](https://debezium.io/documentation/reference/stable/transformations/index.html)  
22. Understanding Kafka Single Message Transform (SMT) \- Blog of Vincent VAUBAN, consulté le septembre 10, 2025, [https://blog.vvauban.com/blog/understanding-kafka-single-message-transform-smt](https://blog.vvauban.com/blog/understanding-kafka-single-message-transform-smt)  
23. How to Implement Debezium SMT (Single Message Transformations)? \- Hevo Data, consulté le septembre 10, 2025, [https://hevodata.com/learn/what-is-debezium-smt/](https://hevodata.com/learn/what-is-debezium-smt/)  
24. Debezium Event Deserialization, consulté le septembre 10, 2025, [https://debezium.io/documentation/reference/stable/integrations/serdes.html](https://debezium.io/documentation/reference/stable/integrations/serdes.html)  
25. Salesforce Connector (Source and Sink) | Confluent Hub, consulté le septembre 10, 2025, [https://www.confluent.io/hub/confluentinc/kafka-connect-salesforce](https://www.confluent.io/hub/confluentinc/kafka-connect-salesforce)  
26. Salesforce Connector (Source and Sink) for Confluent Platform, consulté le septembre 10, 2025, [https://docs.confluent.io/kafka-connectors/salesforce/current/overview.html](https://docs.confluent.io/kafka-connectors/salesforce/current/overview.html)  
27. Subscribing to Platform Events \- Salesforce Developers, consulté le septembre 10, 2025, [https://developer.salesforce.com/docs/atlas.en-us.platform\_events.meta/platform\_events/platform\_events\_subscribe.htm](https://developer.salesforce.com/docs/atlas.en-us.platform_events.meta/platform_events/platform_events_subscribe.htm)  
28. Subscribe to Platform Event Messages with Flows \- Salesforce Developers, consulté le septembre 10, 2025, [https://developer.salesforce.com/docs/atlas.en-us.platform\_events.meta/platform\_events/platform\_events\_subscribe\_flow.htm](https://developer.salesforce.com/docs/atlas.en-us.platform_events.meta/platform_events/platform_events_subscribe_flow.htm)  
29. Subscribe to Platform Event Notifications with Apex Triggers ..., consulté le septembre 10, 2025, [https://developer.salesforce.com/docs/atlas.en-us.platform\_events.meta/platform\_events/platform\_events\_subscribe\_apex.htm](https://developer.salesforce.com/docs/atlas.en-us.platform_events.meta/platform_events/platform_events_subscribe_apex.htm)  
30. How to Subscribe to Platform Events with Apex Triggers : r/salesforce \- Reddit, consulté le septembre 10, 2025, [https://www.reddit.com/r/salesforce/comments/1i0zqgt/how\_to\_subscribe\_to\_platform\_events\_with\_apex/](https://www.reddit.com/r/salesforce/comments/1i0zqgt/how_to_subscribe_to_platform_events_with_apex/)  
31. Salesforce Platform Events — Deep Dive | by Gadige Chakra Dhari | Medium, consulté le septembre 10, 2025, [https://medium.com/@gadige.sfdc/salesforce-platform-events-deep-dive-d9ed9416d405](https://medium.com/@gadige.sfdc/salesforce-platform-events-deep-dive-d9ed9416d405)  
32. Platform Events Developer Guide | Salesforce Developers, consulté le septembre 10, 2025, [https://developer.salesforce.com/docs/atlas.en-us.platform\_events.meta/platform\_events/platform\_events\_intro.htm](https://developer.salesforce.com/docs/atlas.en-us.platform_events.meta/platform_events/platform_events_intro.htm)
