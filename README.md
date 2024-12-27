# Governance for Data-in-Motion

## Pau Harvener, Principal Consultant, Data-Blitz

### Overview
At Data-Blitz, we've observed a growing demand for governance and data quality control in data-in-motion systems. This guide serves as a comprehensive resource for software engineers seeking to create a proof of concept (POC) focused on data-in-motion governance. It includes all necessary artifacts for successfully building and executing such a POC.

POCs involving distributed systems often require in-depth knowledge of each component, which can be challenging given limited time allocations. This guide addresses these challenges with a step-by-step process for designing and executing a Data Governance POC using the Confluent Platform Schema Registry and Data Contracts. All required artifacts are included in the appendices and linked to a Git repository for easy access.

### Definition of Data in Motion
Data in motion refers to data actively moving through a system's pipeline. Wikipedia defines data in motion as:

> Data actively moving through systems, often in real-time.

This paper examines data in motion within stream processing using Kafka. Confluent, through its Enterprise Platform, provides tools for efficiently enforcing data quality and transformation.

### Evolution of Data Governance
Initially, ensuring data quality in motion primarily focused on encryption, often implemented via SSL/TLS. Over time, organizations began developing custom data producers to validate data against standards, leading to more reusable solutions such as the Confluent Schema Registry introduced in 2018. This tool validated data structure, supported schema migration, and enabled seamless integration of new and legacy schemas.

Confluent later introduced Data Contracts, enhancing governance by including data semantics and shifting data quality responsibility to producers. This "shift-left" approach empowers producers to ensure data integrity before reaching consumers.

### Kafka for Governance in Motion
Kafka-based messaging systems are ideal for enforcing data governance in motion due to their role as a central conduit in distributed architectures. Kafka validates data, applies governance policies, and flags discrepancies before data reaches its destination. This centralized approach ensures uniformity and reduces complexity across systems.

### Confluent Platform’s Data Governance Ecosystem
#### Schema Registry
The Confluent Schema Registry manages schemas for Kafka topics, ensuring consistent data formats and enabling seamless schema evolution. Supported schema formats include:
- **Avro**: Compact and supports schema evolution.
- **JSON**: Human-readable but less space-efficient.
- **Protobuf**: Efficient and flexible, with compatibility support.

#### Schema Compatibility Modes
1. **Backward Compatibility**: New schemas can read older data.
2. **Forward Compatibility**: Older schemas can read new data.
3. **Full Compatibility**: Ensures both backward and forward compatibility.

#### Schema Migration Summary
While the Schema Registry supports schema evolution, limitations exist. Defaults in Avro are essential for optional fields, but they may feel restrictive. Confluent’s Data Contracts and migration rules address these challenges.

### Confluent Data Contracts
Data Contracts enhance the Schema Registry by allowing the inclusion of metadata and rule sets in addition to schemas. Components include:
- **Metadata**: Provides context like data lineage, ownership, and compliance guidelines.
- **Rulesets**: Enforce data quality and ensure compatibility during schema evolution.

#### Types of Rulesets
1. **Domain Rules**: Validate current data against domain-specific logic.
2. **Migration Rules**: Enable schema evolution without breaking existing consumers.

### POC Examples
Upcoming examples will use Avro schemas to demonstrate scenarios such as:
- Subjects with complex migration rules.
- Domain validation rules with Dead Letter Queues (DLQ).

These examples will run on the Confluent Platform Enterprise Edition via Docker Compose, requiring access to Docker Hub and a local network. All necessary artifacts, including configurations and schemas, are available in the appendices or the GitHub repository: [GitHub - Data-Blitz Demos](https://github.com/data-blitz-demos/data-in-motion).

### Summary
Kafka messaging systems, combined with Confluent’s governance tools, provide an integrated model for ensuring data integrity and reliability. By leveraging the Schema Registry, Data Contracts, and centralized enforcement, organizations can maintain consistent, high-quality data throughout its lifecycle, from generation to analysis.


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


# Example 2: Data Quality Governance

In the previous example, we demonstrated how to send Avro messages to a specified topic with the Schema Registry ensuring schema compliance. Now, we’ll enhance the pipeline by incorporating **Data Quality Governance**, which evaluates data values and checks their semantic correctness. Since Avro schemas alone do not natively enforce value constraints for individual fields, we’ll leverage **Confluent Platform Data Contracts** to implement these validations.

Continuing from the earlier example, we’ll work with the `order-transaction` schema and introduce **Data Quality Rules** into the Subject's value schema. Confluent Platform Data Contracts allow us to define these rules using either **Google Common Expression Language (CEL)** or **JSONata**. For this example, we’ll use CEL, which is generally faster—an important consideration since every message in the topic must undergo validation.

---

## Data Quality Rule

For the `order-transaction` subject, every message written to the topic must comply with the following rule:

**Rule**: The age of an individual must be greater than 18.

**Rule Definition**:
```json
{
    "ruleSet": {
        "domainRules": [
            {
                "name": "checkForMinors",
                "kind": "CONDITION",
                "type": "CEL",
                "mode": "WRITE",
                "expr": "message.age > 17"
            }
        ]
    }
}
```

If you prefer not to download the JSON object from the Git repository, copy and paste it into a file named `order-transaction-ruleSet-simple.json` and save it in the `EXAMPLE_HOME` directory.

### Structure Breakdown
- The `ruleSet` object contains an array called `domainRules`, where each element represents an individual rule evaluated in sequence.
- This example includes a single rule: `checkForMinors`, which ensures the `age` field is greater than 17.

---

## Setup Steps

### Step 1: Restart Confluent Platform

To ensure a clean environment, restart Confluent Platform:
```bash
docker-compose down
docker-compose up -d
```

**Expected Response** (for `docker-compose down`):
```plaintext
[+] Running 10/10
 ✔ Container control-center       Removed
 ✔ Container ksql-datagen         Removed
 ✔ Container rest-proxy           Removed
 ✔ Container ksqldb-cli           Removed
 ✔ Container ksqldb-server        Removed
 ✔ Container connect              Removed
 ✔ Container schema-registry      Removed
 ✔ Container broker               Removed
 ✔ Container zookeeper            Removed
 ✔ Network confluent-770_default  Removed
```

### Step 2: Register the Schema

Re-register the `order-transaction.avsc` schema with the Schema Registry:
```bash
jq -n --rawfile schema order-transaction.avsc '{schema: $schema}' | \
curl http://localhost:8081/subjects/order-transactions-value/versions --json @-
```

**Expected Response**:
```json
{"id":1}
```

### Step 3: Add the RuleSet

Add the rule set to the Subject’s value schema:
```bash
curl http://localhost:8081/subjects/order-transactions-value/versions \
  --json @order-transaction-ruleSet-simple.json
```

**Expected Response**:
```json
{"id":2,"version":2,"ruleSet":{"domainRules":[{"name":"checkForMinors","kind":"CONDITION","mode":"WRITE","type":"CEL","expr":"message.age > 17","disabled":false}]},"schema":"..."}
```

---

## Testing the Rule

### Step 1: Start the Kafka Consumer

Start the Kafka Avro console consumer:
```bash
./bin/kafka-avro-console-consumer \
  --bootstrap-server localhost:9092 \
  --from-beginning \
  --topic order-transactions \
  --property schema.registry.url=http://localhost:8081
```

**Expected Response**:
```plaintext
use.latest.with.metadata = null
use.schema.id = -1
value.subject.name.strategy = class io.confluent.kafka.serializers.subject.TopicNameStrategy
```

### Step 2: Start the Kafka Producer

Start the Kafka Avro console producer:
```bash
./bin/kafka-avro-console-producer \
  --topic order-transactions \
  --broker-list localhost:9092 \
  --property value.schema.id=2 \
  --property bootstrap.servers=localhost:9092
```

**Expected Response**:
```plaintext
use.latest.version = false
use.latest.with.metadata = null
use.schema.id = -1
value.subject.name.strategy = class io.confluent.kafka.serializers.subject.TopicNameStrategy
```

### Step 3: Test with Valid Data

Send a valid message:
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
  "creditCardNumberType": "Mastercard",
  "creditCardNumber": "2541123132728249"
}
```

**Expected Result**:
- Producer: No errors.
- Consumer: The message appears in the topic.

### Step 4: Test with Invalid Data

Send an invalid message (age < 18):
```json
{
  "transactionId": "1f04e109-73a8-495c-a7b9-674c7779a130",
  "productId": "4304360364601",
  "price": 87.325,
  "productDescripton": "Stainless steel garden trowel with ergonomic handle.",
  "timestamp": 1729184505,
  "firstName": "Doug",
  "lastName": "Smith",
  "email": "doug.smith@yahoo.com",
  "gender": "Male",
  "age": 13,
  "address": "9900 Curtis Field Suite 242",
  "city": "West Katieland",
  "state": "VA",
  "zipCode": "41744",
  "creditCardNumberType": "Visa",
  "creditCardNumber": "5541123132728247"
}
```

**Expected Response**:
```plaintext
Caused by: org.apache.kafka.common.errors.SerializationException: Rule failed: checkForMinors
...
Caused by: io.confluent.kafka.schemaregistry.rules.RuleConditionException: Expr failed: 'message.age > 17'
```

The producer throws an exception, indicating that the `checkForMinors` rule failed. The invalid message is not written to the topic.

---

## Using a Dead Letter Queue (DLQ)

To improve error handling, we can configure a Dead Letter Queue (DLQ). This allows failed messages to be written to a separate topic for further analysis.

### Updated Rule with DLQ
```json
{
    "ruleSet": {
        "domainRules": [
            {
                "name": "checkForMinors",
                "kind": "CONDITION",
                "type": "CEL",
                "mode": "WRITE",
                "expr": "message.age > 17",
                "params": {
                    "dlq.topic": "order-transactions-dlq"
                },
                "onFailure": "DLQ"
            }
        ]
    }
}
```

Save this updated RuleSet as `order-transaction-ruleSet-dlq.json` and update the value schema:
```bash
curl http://localhost:8081/subjects/order-transactions-value/versions \
  --json @order-transaction-ruleSet-dlq.json
```

### Testing with DLQ

Re-run the producer with invalid data. Instead of throwing an exception, the message will be written to the DLQ.

Check the `order-transactions-dlq` topic in Control Center or through the consumer:
```bash
./bin/kafka-avro-console-consumer \
  --bootstrap-server localhost:9092 \
  --from-beginning \
  --topic order-transactions-dlq \
  --property schema.registry.url=http://localhost:8081
```

**Observation**:
- The invalid message is present in the DLQ.
- The message header contains metadata identifying the failed rule and the reason for rejection.

---

This example demonstrates how Data Quality Governance ensures adherence to business rules, leveraging DLQs to manage invalid data effectively.





# Example 3: Complex Data Quality Governance

In this example, we will enforce semantic governance around the use of credit cards. The `order-transaction.avsc` schema will be used to ensure compliance with specific constraints. The governance rules will validate data based on customer age and credit card information as follows:

## Constraints:
1. Customers must be 18 years old or older.
2. Supported credit card types:
   - **Visa**:
     - The card number must start with the digit `5`.
     - The card number must be exactly 16 digits long.
   - **Mastercard**:
     - The card number must start with the digit `2`.
     - The card number must be exactly 16 digits long.
   - **AMEX**:
     - The card number must start with the digit `3`.
     - The card number must be exactly 15 digits long.

---

## Setup Steps

### Restart Docker-Compose
To initialize the environment, restart Docker-Compose. Use one of the following commands:
```bash
docker-compose down
docker-compose up -d
```
Alternatively, use Docker-Compose plugins in Visual Studio Code if preferred.

### Register the Avro Schema
Register the `order-transaction.avsc` schema with the Confluent Schema Registry:
```bash
jq -n --rawfile schema order-transaction.avsc '{schema: $schema}' | \
curl http://localhost:8081/subjects/order-transactions-value/versions --json @-
```

### Rule Set Setup
If the JSON object is not downloaded from the Git repository, copy and paste the JSON object in Appendix 3 into a file named `order-transaction-ruleSet-complex.json` within the `EXAMPLE_HOME` directory. This RuleSet will implement the constraints using Google Common Expression Language (CEL).

Add the RuleSet to the `order-transactions-value` subject by executing the following command:
```bash
curl http://localhost:8081/subjects/order-transactions-value/versions \
  --json @order-transaction-ruleSet-complex.json
```

---

## Starting Kafka Console Tools

### Kafka Producer
Start the Kafka Avro console producer:
```bash
./bin/kafka-avro-console-producer \
  --topic order-transactions \
  --broker-list localhost:9092 \
  --property value.schema.id=2 \
  --property bootstrap.servers=localhost:9092 \
  --property dlq.auto.flush=true
```

### Kafka Consumer
Start the Kafka Avro console consumer in a new terminal:
```bash
./bin/kafka-avro-console-consumer \
  --bootstrap-server localhost:9092 \
  --from-beginning \
  --topic order-transactions \
  --property schema.registry.url=http://localhost:8081
```

---

## RuleSet Description
The RuleSet file `order-transaction-ruleSet-complex.json` contains the following JSON object:

```json
{
  "ruleSet": {
    "domainRules": [
      {
        "name": "customers_under_age_of_18_not_supported",
        "kind": "CONDITION",
        "type": "CEL",
        "mode": "WRITE",
        "expr": "message.age >= 18",
        "params": {
          "dlq.topic": "order-transactions-dlq"
        },
        "onFailure": "DLQ"
      },
      {
        "name": "unsupported_credit_card_type",
        "kind": "CONDITION",
        "type": "CEL",
        "mode": "WRITE",
        "expr": "message.creditCardNumberType in [\"AMEX\", \"Visa\", \"Mastercard\"]",
        "params": {
          "dlq.topic": "order-transactions-dlq"
        },
        "onFailure": "DLQ"
      },
      {
        "name": "visa_card_number_is_not_16_digits_long",
        "kind": "CONDITION",
        "type": "CEL",
        "mode": "WRITE",
        "expr": "message.creditCardNumberType == \"Visa\" ? size(message.creditCardNumber) == 16:true",
        "params": {
          "dlq.topic": "order-transactions-dlq"
        },
        "onFailure": "DLQ"
      },
      {
        "name": "visa_card_number_first_digit_is_not_5",
        "kind": "CONDITION",
        "type": "CEL",
        "mode": "WRITE",
        "expr": "message.creditCardNumberType == \"Visa\" ? message.creditCardNumber.matches(\"^5[0-9]{15}$\"):true",
        "params": {
          "dlq.topic": "order-transactions-dlq"
        },
        "onFailure": "DLQ"
      }
    ]
  }
}
```

### Rule Breakdown
1. **Age Validation**: Ensures customers are at least 18 years old. Violations are routed to the Dead Letter Queue (DLQ).
2. **Credit Card Type Validation**: Only Visa, Mastercard, or AMEX are accepted. Unsupported types are sent to the DLQ.
3. **Visa Card Length**: Ensures Visa card numbers are 16 digits long.
4. **Visa Card Prefix**: Ensures Visa card numbers start with the digit `5`.

---

## Testing Rules

### Valid Message
Send a valid transaction with a Visa card meeting all requirements:
```json
{
  "transactionId": "1f04e109-73a8-495c-a7b9-674c7779a130",
  "productId": "4304360364601",
  "price": 874.34,
  "productDescripton": "Stainless steel garden trowel with ergonomic handle.",
  "timestamp": 1729184505,
  "firstName": "Sam",
  "lastName": "Murray",
  "email": "sam.murray@yahoo.com",
  "gender": "male",
  "age": 54,
  "address": "9900 Curtis Field Suite 242",
  "city": "West Katieland",
  "state": "VA",
  "zipCode": "41744",
  "creditCardNumberType": "Visa",
  "creditCardNumber": "5541123132728247"
}
```

**Expected Result**:
- Producer: No errors.
- Consumer: The message appears in the topic.

### Invalid Messages

1. **Age Validation Failure**:
   ```json
   {
     "transactionId": "1f04e109-73a8-495c-a7b9-674c7779a130",
     "productId": "4304360364601",
     "price": 874.34,
     "productDescripton": "Stainless steel garden trowel with ergonomic handle.",
     "timestamp": 1729184505,
     "firstName": "Doug",
     "lastName": "Smith",
     "email": "doug.smith@yahoo.com",
     "gender": "Male",
     "age": 16,
     "address": "123 Main Street",
     "city": "Springfield",
     "state": "IL",
     "zipCode": "62704",
     "creditCardNumberType": "Visa",
     "creditCardNumber": "5541123132728247"
   }
   ```
   **Explanation**: The customer age is below 18.

   **Expected Result**:
   - Producer: Error: `Rule failed: customers_under_age_of_18_not_supported`.

2. **Visa Card Number Length Failure**:
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
     "creditCardNumberType": "Visa",
     "creditCardNumber": "554112313272824"
   }
   ```
   **Explanation**: The Visa card number is not 16 digits long.

   **Expected Result**:
   - Producer: Error: `Rule failed: visa_card_number_is_not_16_digits_long`.

---

## Dead Letter Queue (DLQ) Observations
All failing messages are routed to the DLQ for debugging. The message header contains metadata about the failed rules, such as the name of the rule and additional context for troubleshooting.

---


# Example 3: Complex Data Quality Governance

In this example, we enforce semantic governance around the use of credit cards using the `order-transaction.avsc` schema. The following constraints will ensure data quality:

## Constraints
1. Customers must be 18 years old or older.
2. Supported credit card types:
   - **Visa**:
     - Leading digit must be `5`.
     - Card number must be exactly 16 digits long.
   - **MasterCard**:
     - Leading digit must be `2`.
     - Card number must be exactly 16 digits long.
   - **AMEX**:
     - Leading digit must be `3`.
     - Card number must be exactly 15 digits long.

---

## Setup Steps

### Step 1: Restart Docker Compose

Restart Docker Compose to initialize the environment:
```bash
docker-compose down
docker-compose up -d
```

### Step 2: Register the Schema

Register the `order-transaction.avsc` schema with the Confluent Schema Registry:
```bash
jq -n --rawfile schema order-transaction.avsc '{schema: $schema}' | \
curl http://localhost:8081/subjects/order-transactions-value/versions --json @-
```

### Step 3: Create and Register RuleSet

1. Copy and paste the JSON RuleSet (below) into a file named `order-transaction-ruleSet-complex.json` in the `EXAMPLE_HOME` directory.

```json
{
    "ruleSet": {
        "domainRules": [
            {
                "name": "customers_under_age_of_18_not_supported",
                "kind": "CONDITION",
                "type": "CEL",
                "mode": "WRITE",
                "expr": "message.age >= 18",
                "params": {
                    "dlq.topic": "order-transactions-dlq"
                },
                "onFailure": "DLQ"
            },
            {
                "name": "unsupported_credit_card_type",
                "kind": "CONDITION",
                "type": "CEL",
                "mode": "WRITE",
                "expr": "message.creditCardNumberType in [\"AMEX\", \"Visa\", \"MasterCard\"]",
                "params": {
                    "dlq.topic": "order-transactions-dlq"
                },
                "onFailure": "DLQ"
            },
            {
                "name": "visa_card_number_is_not_16_digits_long",
                "kind": "CONDITION",
                "type": "CEL",
                "mode": "WRITE",
                "expr": "message.creditCardNumberType == \"Visa\" ? size(message.creditCardNumber) == 16:true",
                "params": {
                    "dlq.topic": "order-transactions-dlq"
                },
                "onFailure": "DLQ"
            },
            {
                "name": "visa_card_number_first_digit_is_not_5",
                "kind": "CONDITION",
                "type": "CEL",
                "mode": "WRITE",
                "expr": "message.creditCardNumberType == \"Visa\" ? message.creditCardNumber.matches(\"^5[0-9]{15}$\"):true",
                "params": {
                    "dlq.topic": "order-transactions-dlq"
                },
                "onFailure": "DLQ"
            }
        ]
    }
}
```

2. Add the RuleSet to the `order-transactions-value` subject:
```bash
curl http://localhost:8081/subjects/order-transactions-value/versions \
  --json @order-transaction-ruleSet-complex.json
```

---

## Start Kafka Tools

### Start Kafka Producer

Run the following command to start the Kafka producer:
```bash
./bin/kafka-avro-console-producer \
  --topic order-transactions \
  --broker-list localhost:9092 \
  --property value.schema.id=2 \
  --property bootstrap.servers=localhost:9092 \
  --property dlq.auto.flush=true
```

### Start Kafka Consumer

Run the following command to start the Kafka consumer:
```bash
./bin/kafka-avro-console-consumer \
  --bootstrap-server localhost:9092 \
  --from-beginning --topic order-transactions \
  --property schema.registry.url=http://localhost:8081
```

---

## Testing Rules

### Valid Message

Send a valid Visa transaction:
```json
{
  "transactionId": "1f04e109-73a8-495c-a7b9-674c7779a130",
  "productId": "4304360364601",
  "price": 874.34,
  "productDescripton": "Stainless steel garden trowel with ergonomic handle.",
  "timestamp": 1729184505,
  "firstName": "Sam",
  "lastName": "Murray",
  "email": "sam.murray@yahoo.com",
  "gender": "male",
  "age": 54,
  "address": "9900 Curtis Field Suite 242",
  "city": "West Katieland",
  "state": "VA",
  "zipCode": "41744",
  "creditCardNumberType": "Visa",
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
    "firstName": "Sam",
    "lastName": "Murray",
    "email": "sam.murray@yahoo.com",
    "gender": "male",
    "age": 54,
    "address": "9900 Curtis Field Suite 242",
    "city": "West Katieland",
    "state": "VA",
    "zipCode": "41744",
    "creditCardNumberType": "Visa",
    "creditCardNumber": "5541123132728247"
  }
  ```

### Invalid Messages

1. **Visa Card - Invalid Starting Digit:**
   ```json
   {
     "transactionId": "1f04e109-73a8-495c-a7b9-674c7779a130",
     "productId": "4304360364601",
     "price": 874.34,
     "productDescripton": "Stainless steel garden trowel with ergonomic handle.",
     "timestamp": 1729184505,
     "firstName": "Sam",
     "lastName": "Murray",
     "email": "sam.murray@yahoo.com",
     "gender": "male",
     "age": 54,
     "address": "9900 Curtis Field Suite 242",
     "city": "West Katieland",
     "state": "VA",
     "zipCode": "41744",
     "creditCardNumberType": "Visa",
     "creditCardNumber": "8541123132728247"
   }
   ```

   **Expected Response:**
   ```plaintext
   Rule failed: visa_card_number_first_digit_is_not_5
   ```

2. **Visa Card - Invalid Length:**
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
     "creditCardNumberType": "Visa",
     "creditCardNumber": "554112313272824"
   }
   ```

   **Expected Response:**
   ```plaintext
   Rule failed: visa_card_number_is_not_16_digits_long
   ```

---

## Observations

Faulty messages are routed to the Dead Letter Queue (DLQ) for debugging. The DLQ contains both the invalid message and metadata about the rule(s) that failed. This ensures robust governance and traceability in your pipeline.


# Example 4: Data Transformation Governance

Data transformation is a critical use case in streaming data pipelines. By adopting a "shift left" approach, we aim to apply data transformations as early in the pipeline as possible. This example demonstrates how to govern data transformations using the Confluent Platform Data Contracts framework. Specifically, we will introduce a schema migration that forces a breaking change, using AVRO as the messaging format for streaming data.

## Overview
The Confluent Platform Data Contracts framework supports breaking schema changes by partitioning schema versions into compatibility groups. This capability ensures that both legacy and updated consumers can process messages correctly, regardless of schema changes.

To achieve this, we use migration rules implemented with JSONata, a lightweight query and transformation language for JSON data. These migration rules enable seamless schema compatibility across different versions, ensuring smooth communication between producers and consumers even when schema-breaking changes occur.

### Migration Rules
In this example, two primary migration rules are defined:

1. **UPGRADE Rule**: Allows new consumers expecting the current schema to read messages from older schema versions.
2. **DOWNGRADE Rule**: Allows older consumers expecting the previous schema to read messages using the current schema.

If the system plans to upgrade all consumers to the latest schema version, the DOWNGRADE rule can be omitted.

### JSONata `$sift()` Function
We leverage the JSONata `$sift()` function to dynamically handle schema changes. This function:
- Removes a field by its name.
- Adds a new field with a different name.

This ensures that schema changes, such as removing attributes or renaming fields, are handled dynamically, enabling both old and new consumers to process messages without disruption.

### Example Migration Rules
```json
"migrationRules": [
    {
      "name": "changeFirstNameToFirst_name",
      "kind": "TRANSFORM",
      "type": "JSONATA",
      "mode": "UPGRADE",
      "expr": "$merge([$sift($, function($v, $k) {$k != 'firstName'}), {'first_name': $.firstName}])"
    },
    {
      "name": "changeFirst_nameToFirstName",
      "kind": "TRANSFORM",
      "type": "JSONATA",
      "mode": "DOWNGRADE",
      "expr": "$merge([$sift($, function($v, $k) {$k != 'first_name'}), {'firstName': $.first_name}])"
    }
]
```

### Metadata Example
The metadata controlling the major version for compatibility groups:
```json
{
  "metadata": {
    "compatibility_group": 2
  }
}
```

### Diagram Explanation
This example involves two Kafka producers and two consumers:
1. **Producers**:
   - Producer 1 sends messages using the original schema (`order-transactions`).
   - Producer 2 sends messages using the new schema (`order-messages-migration`).
2. **Consumers**:
   - Consumer 1 expects messages conforming to the old schema (Compatibility Group 1).
   - Consumer 2 expects messages conforming to the new schema (Compatibility Group 2).

### Steps
1. **Restart Environment**: Initialize the environment by restarting `docker-compose`.
2. **Run Example 3**: Re-establish the existing pipeline.
3. **Register Schema**:
   - Register the original schema (`order-transactions-value`).
   - Add migration rules to handle breaking changes.
4. **Validate Results**:
   - Verify message compatibility for both consumers.
   - Test UPGRADE and DOWNGRADE rules.

### Commands
#### Register Original Schema
```bash
jq -n --rawfile schema order-transaction.avsc "{schema: $schema}" | \
  curl http://localhost:8081/subjects/order-transactions-value/versions --json @-
```

#### Create Migration Rules
```bash
curl http://localhost:8081/config/order-transactions-value \
  -X PUT --json "{ \"compatibilityGroup\": \"compatibility_group\" }"
```

#### Start Consumers
- **Consumer 1** (Original Schema):
  ```bash
  ./bin/kafka-avro-console-consumer \
    --topic order-transactions \
    --bootstrap-server localhost:9092 \
    --property auto.register.schemas=false
  ```

- **Consumer 2** (New Schema):
  ```bash
  ./bin/kafka-avro-console-consumer \
    --topic order-transactions \
    --bootstrap-server localhost:9092 \
    --property auto.register.schemas=false \
    --property use.latest.version=true
  ```

#### Start Producers
- **Producer 1** (Original Schema):
  ```bash
  ./bin/kafka-avro-console-producer \
    --topic order-transactions \
    --broker-list localhost:9092 \
    --property value.schema.id=1
  ```

- **Producer 2** (New Schema):
  ```bash
  ./bin/kafka-avro-console-producer \
    --topic order-transactions \
    --broker-list localhost:9092 \
    --property value.schema.id=2
  ```

### Summary
Using migration rules in the Confluent Platform Data Contracts framework enables flexible, rule-based data transformation governance. This approach ensures compatibility across schema versions, even with breaking changes, providing robust support for evolving data pipelines.
