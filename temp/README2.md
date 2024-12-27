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

