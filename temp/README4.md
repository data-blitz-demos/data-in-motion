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
