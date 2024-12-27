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

