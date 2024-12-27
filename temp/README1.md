# Example 1: Data Structural Governance

This example demonstrates how to send AVRO messages to a specified topic, with the Schema Registry enforcing the schema structure. Messages that do not conform will cause the producer to fail, illustrating a basic form of data governance. This serves as the baseline for additional examples, each introducing more sophisticated forms of governance for data in motion.

---

## Setup Requirements

### Prerequisites

1. **Install Docker:**
   Download Docker Desktop from [Docker's official site](https://www.docker.com/products/docker-desktop).

2. **Install curl:**
   Download curl from [curl's official site](https://curl.se/download.html).
   - **Mac Users**: Use Brew to install curl if preferred.

3. **Install jq:**
   Download jq, a command-line tool for parsing JSON data, from [jq's official site](https://jqlang.github.io/jq/download/).
   - **Mac Users**: Use Brew to install jq if preferred.

4. **Install Java:**
   Download and install Java from [Java's official site](https://www.java.com/en/).

5. **Install Confluent Platform:**
   Download the Confluent Platform and unzip it into a directory of your choice (referred to as `EXAMPLE_HOME`). Follow the [installation guide](https://docs.confluent.io/platform/current/installation/installing_cp/zip-tar.html).

6. (Optional) **Install Visual Studio Code:**
   Download and install Visual Studio Code from [Visual Studio Code's official site](https://code.visualstudio.com/download).

---

## Deploying Confluent Platform with Docker Compose

1. **Setup Docker Compose File:**
   If not downloaded, copy the content from Appendix 1 into a file named `docker-compose.yml` and save it in the `EXAMPLE_HOME` directory.

2. **Start Docker Compose:**
   Run the following command to start the Confluent Platform ecosystem:
   ```bash
   docker-compose up -d
   ```

   **Expected Response:**
   ```plaintext
   [+] Running 9/10
    ⠏ Network confluent-770_default  Created
               4.0s
    ✔ Container zookeeper            Started
               1.3s
    ✔ Container broker               Started
               1.6s
    ✔ Container schema-registry      Started
               1.8s
    ✔ Container rest-proxy           Started
               2.6s
    ✔ Container connect              Started
               2.5s
    ✔ Container ksqldb-server        Started
               2.8s
    ✔ Container ksqldb-cli           Started
               3.5s
    ✔ Container control-center       Started
               3.5s
    ✔ Container ksql-datagen         Started
               3.5s
   ```

3. **Verify Control Center:**
   Open your browser and navigate to:
   ```
   http://localhost:9021/clusters
   ```
   - If the cluster appears as "Unhealthy," click "Overview" and then "Brokers" in the left-hand menu to refresh the status to "Healthy."

---

## Registering the AVRO Schema

1. **Setup the AVRO Schema File:**
   If not downloaded, copy the content from Appendix 2 into a file named `order-transaction.avsc` and save it in the `EXAMPLE_HOME` directory.

2. **Register the Schema:**
   Run the following command to add the `order-transaction` schema to the Schema Registry:
   ```bash
   jq -n --rawfile schema order-transaction.avsc '{schema: $schema}' | \
   curl http://localhost:8081/subjects/order-transactions-value/versions --json @-
   ```

   **Expected Response:**
   ```json
   {"id":1}
   ```

---

## Starting Kafka Console Tools

### Start Kafka Consumer
Run the Kafka Avro console consumer in a new terminal:
```bash
./bin/kafka-avro-console-consumer \
  --bootstrap-server localhost:9092 \
  --from-beginning \
  --topic order-transactions \
  --property schema.registry.url=http://localhost:8081
```

**Expected Response:**
```plaintext
use.latest.with.metadata = null
use.schema.id = -1
value.subject.name.strategy = class io.confluent.kafka.serializers.subject.TopicNameStrategy
```

### Start Kafka Producer
Run the Kafka Avro console producer in a new terminal:
```bash
./bin/kafka-avro-console-producer \
  --topic order-transactions \
  --broker-list localhost:9092 \
  --property schema.registry.url=http://localhost:8081 \
  --property value.schema.id=1
```

**Expected Response:**
```plaintext
use.latest.version = false
use.latest.with.metadata = null
use.schema.id = -1
value.subject.name.strategy = class io.confluent.kafka.serializers.subject.TopicNameStrategy
```

---

## Producing and Consuming Messages

### Valid Message
Send the following valid message via the Kafka producer:
```json
{
  "transactionId": "1f04e109-73a8-495c-a7b9-674c7779a130",
  "productId": "4304360364601",
  "price": 874.34,
  "productDescripton": "Stainless steel garden trowel with ergonomic handle.",
  "timestamp": 1729184505,
  "firstName": "Amanda",
  "lastName": "Murray",
  "email": "amanda.murray@yahoo.com",
  "gender": "Female",
  "age": 67,
  "address": "9900 Curtis Field Suite 242",
  "city": "West Katieland",
  "state": "VA",
  "zipCode": "41744",
  "creditCardNumberType": "AMEX",
  "creditCardNumber": "5541123132728247"
}
```

**Expected Result:**
- Producer: No errors.
- Consumer:
  ```json
  {
    "transactionId": "1f04e109-73a8-495c-a7b9-674c7779a130",
    "productId": "4304360364601",
    "price": 874.34,
    "productDescripton": "Stainless steel garden trowel with ergonomic handle.",
    "timestamp": 1729184505,
    "firstName": "Amanda",
    "lastName": "Murray",
    "email": "amanda.murray@yahoo.com",
    "gender": "Female",
    "age": 67,
    "address": "9900 Curtis Field Suite 242",
    "city": "West Katieland",
    "state": "VA",
    "zipCode": "41744",
    "creditCardNumberType": "AMEX",
    "creditCardNumber": "5541123132728247"
  }
  ```

### Invalid Message
Send the following invalid message (non-string `productId`):
```json
{
  "transactionId": "1f04e109-73a8-495c-a7b9-674c7779a130",
  "productId": 4304360364601,
  "price": 874.34,
  "productDescripton": "Stainless steel garden trowel with ergonomic handle.",
  "timestamp": 1729184505,
  "firstName": "Sam",
  "lastName": "Murray",
  "email": "sam.murray@yahoo.com",
  "gender": "Male",
  "age": 12,
  "address": "9900 Curtis Field Suite 242",
  "city": "West Katieland",
  "state": "VA",
  "zipCode": "41744",
  "creditCardNumberType": "Visa",
  "creditCardNumber": "5541123132728247"
}
```

**Expected Result:**
- Producer fails with the following exception:
  ```plaintext
  Caused by: org.apache.avro.AvroTypeException: Expected string. Got VALUE_NUMBER_INT
  ```

---

## Observing Messages in Control Center

1. Open your browser and navigate to:
   ```
   http://localhost:9021/clusters/overview
   ```

2. Click the "Topics" link in the left menu.

3. Select the `order-transactions` topic.

4. View the messages and schema to confirm data correctness.

---

## Summary

In this example, we:
- Created and registered an AVRO schema with the Schema Registry.
- Produced schema-compliant messages to the `order-transactions` topic.
- Verified message validity using the Schema Registry on both the producer and consumer sides.
- Observed schema-enforced governance in action, ensuring data consistency and correctness.

This "shift-left" approach validates data early in the pipeline, preventing invalid data from propagating through the system. Future examples will extend this foundation by adding advanced governance capabilities.

