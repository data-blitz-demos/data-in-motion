Here’s the full and detailed `README.md`, including **detailed step-by-step instructions** for running all examples (1 through 4):

```markdown
# Order-Transactions: Governance for Data in Motion

## Overview
This project demonstrates governance for data in motion using Kafka, Confluent Schema Registry, and data contracts. The examples focus on validating data quality, enforcing schema compliance, handling invalid messages via a Dead Letter Queue (DLQ), and migrating schemas with minimal disruption. All examples work with the `order-transactions` Kafka topic.

**About Data-Blitz:**  
This project is brought to you by **Data-Blitz**, a consultancy specializing in real-time, event-driven data processing and advanced data governance solutions. With expertise in designing scalable architectures, Data-Blitz empowers businesses to unlock the full potential of their data in motion. This repository serves as a hands-on guide to showcase practical implementations of the concepts we advocate in real-world use cases.

---

## About Confluent Platform Data Contracts

**Data Contracts** are a powerful feature of the **Confluent Platform** that extends beyond basic schema management by enabling precise control over data governance. Data Contracts allow you to define not only the schema of your data but also rules for validating and transforming data as it flows through Kafka. These contracts ensure that producers and consumers adhere to agreed-upon structures and business logic, reducing errors and increasing reliability.

---

## Prerequisites
To run the POC examples, ensure you have the following installed and configured:
- **Docker and Docker Compose**: To deploy Kafka, Schema Registry, and related components.  
  [Download Docker](https://www.docker.com/products/docker-desktop)  
- **cURL**: For interacting with REST endpoints.  
  [Download cURL](https://curl.se/download.html)  
- **jq**: Command-line tool for parsing JSON.  
  [Download jq](https://jqlang.github.io/jq/download/)  
- **Java (JDK 11 or higher)**: For Avro console tools.  
  [Download Java](https://java.com/en/)  
- **Kafka and Confluent Platform**: Installed locally or via Docker Compose.  
  [Install Confluent](https://docs.confluent.io/platform/current/installation/installing_cp/zip-tar.html)  

---

## Common Setup
1. Clone the repository:
   ```bash
   git clone https://github.com/your-org/order-transactions.git
   cd order-transactions
   ```

2. Start the Confluent Platform services using Docker Compose:
   ```bash
   docker-compose up -d
   ```

3. Verify that Kafka and Schema Registry are running:
   - Kafka Broker: `localhost:9092`
   - Schema Registry: `http://localhost:8081`

---

## POC Examples

### **Example 1: Schema Validation with Avro**
**Overview:**  
This example demonstrates how to validate schema compliance for messages produced to the `order-transactions` Kafka topic. It shows how to register an Avro schema with the Schema Registry and enforce that only schema-compliant messages can be successfully written to the topic.

**Steps to Run:**
1. Save the schema as `order-transaction.avsc`:
   ```json
   {
     "type": "record",
     "name": "OrderTransaction",
     "fields": [
       {"name": "transactionId", "type": "string"},
       {"name": "productId", "type": "string"},
       {"name": "price", "type": "float"},
       {"name": "timestamp", "type": "long"},
       {"name": "firstName", "type": "string"},
       {"name": "lastName", "type": "string"}
     ]
   }
   ```

2. Register the schema in the Schema Registry:
   ```bash
   jq -n --rawfile schema order-transaction.avsc '{schema: $schema}' | \
   curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
   --url http://localhost:8081/subjects/order-transactions-value/versions \
   --data @-
   ```

3. Produce a valid message:
   ```bash
   ./bin/kafka-avro-console-producer \
   --topic order-transactions \
   --broker-list localhost:9092 \
   --property value.schema.id=1
   ```
   Enter the following payload:
   ```json
   {"transactionId":"123","productId":"456","price":100.0,"timestamp":1635600000,"firstName":"John","lastName":"Doe"}
   ```

4. Consume the message:
   ```bash
   ./bin/kafka-avro-console-consumer \
   --bootstrap-server localhost:9092 \
   --topic order-transactions \
   --from-beginning \
   --property schema.registry.url=http://localhost:8081
   ```

**Expected Result:**  
The consumer displays the same message you produced.

---

### **Example 2: Data Quality Enforcement**
**Overview:**  
This example focuses on enforcing data quality rules using domain rules within a data contract. Specifically, it ensures that the `age` field in messages is greater than 17, preventing invalid data from being written to the topic.

**Steps to Run:**
1. Save the rule set as `order-transaction-ruleSet.json`:
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

2. Add the rule set to the Schema Registry:
   ```bash
   curl -X POST -H "Content-Type: application/vnd.schemaregistry.rules+json" \
   --url http://localhost:8081/subjects/order-transactions-value/versions/1/rules \
   --data @order-transaction-ruleSet.json
   ```

3. Produce a valid message:
   ```bash
   ./bin/kafka-avro-console-producer \
   --topic order-transactions \
   --broker-list localhost:9092 \
   --property value.schema.id=1
   ```
   Enter a valid payload:
   ```json
   {"transactionId":"124","productId":"457","price":50.0,"timestamp":1635600001,"firstName":"Jane","lastName":"Doe","age":18}
   ```

4. Produce an invalid message:
   ```json
   {"transactionId":"125","productId":"458","price":60.0,"timestamp":1635600002,"firstName":"Jim","lastName":"Beam","age":16}
   ```

**Expected Result:**  
- The valid message is successfully written to the topic.  
- The invalid message fails, and the producer throws an error.

---

### **Example 3: Dead Letter Queue (DLQ)**
**Overview:**  
This example demonstrates how to handle invalid messages that fail domain rule validation. Instead of rejecting such messages outright, they are routed to a Dead Letter Queue (DLQ) for later inspection and processing.

**Steps to Run:**
1. Update the rule set to include a DLQ:
   Save as `order-transaction-ruleSet-dlq.json`:
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

   Add it to the Schema Registry:
   ```bash
   curl -X POST -H "Content-Type: application/vnd.schemaregistry.rules+json" \
   --url http://localhost:8081/subjects/order-transactions-value/versions/2/rules \
   --data @order-transaction-ruleSet-dlq.json
   ```

2. Produce an invalid message:
   ```bash
   ./bin/kafka-avro-console-producer \
   --topic order-transactions \
   --broker-list localhost:9092 \
   --property value.schema.id=2
   ```
   Enter:
   ```json
   {"transactionId":"126","productId":"459","price":100.0,"timestamp":1635600003,"firstName":"Anna","lastName":"Smith","age":16}
   ```

3. Consume messages from the DLQ:
   ```bash
   ./bin/kafka-console-consumer \
   --topic order-transactions-dlq \
   --bootstrap-server localhost:9092 \
   --from-beginning
   ```

**Expected Result:**  
The invalid message is found in the DLQ.

---

### **Example 4: Migration Rules**
**Overview:**  
This example showcases how to use migration rules to handle schema evolution without breaking existing producers or consumers.

**Steps to Run:**
1. Save the migration rule as `order-transaction-migrationRule.json`:
   ```json
   {
     "ruleSet": {
       "migrationRules": [
         {
           "name": "defaultAge",
           "kind": "TRANSFORM",
           "type": "CEL",
           "mode": "READ",
           "expr": "message.age == null ? {age: 18} : message"
         }
       ]
     }
   }
   ```

2. Add it to the Schema Registry:
   ```bash
   curl -X POST -H "Content-Type: application/vnd.schemaregistry.rules+json" \
   --url http://localhost:8081/subjects/order-transactions-value/versions/3/rules \
   --data @order-transaction-migrationRule.json
   ```

3. Produce a message without the `age` field:
   ```bash
   ./bin/kafka-avro-console-producer \
   --topic order-transactions \
   --broker-list localhost:9092 \
   --property value.schema.id=3
   ```
   Enter:
   ```json
   {"transactionId":"127","productId":"460","price":150.0,"timestamp":1635600004,"firstName":"Mark","lastName":"Lee"}
   ```

4. Consume the message:
   ```bash
   ./bin/kafka-avro-console-consumer \
   --bootstrap-server localhost:9092 \
   --topic order-transactions \
   --from-beginning \
   --property schema.registry.url=http://localhost:8081
   ```

**Expected Result:**  
The consumer adds the `age` field with the default value:
```json
{"transactionId":"127","productId":"460","price":150.0,"timestamp":1635600004,"firstName":"Mark","lastName":"Lee","age":18}
```

---

## Troubleshooting
- **Schema not found**: Verify the schema is registered with the correct subject.
- **Validation failures**: Check the rule set for typos or misconfigurations.
- **Connection issues**: Ensure all services are running and accessible.

---

## Additional Resources
- [Confluent Schema Registry Documentation](https://docs.confluent.io/current/schema-registry/index.html)
- [Confluent Data Contracts Documentation](https://docs.confluent.io/platform/current/schema-registry/data-contracts.html)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Avro Schema Specification](https://avro.apache.org/docs/current/spec.html)
```
