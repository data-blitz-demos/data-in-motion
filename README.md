

---

# Governance with Data in Motion

## Overview

This repository accompanies the white paper *Governance with Data in Motion. The paper serves as a comprehensive guide for software engineers to create a Proof of Concept (POC) for governance in stream processing systems, particularly focusing on **data in motion** using the **Confluent Platform** and **Kafka**.

It outlines the importance of enforcing data governance and quality control in real-time pipelines and provides step-by-step instructions, examples, and the necessary artifacts for building robust data governance mechanisms.

---

## Key Features

### Concepts and Components
1. **Data in Motion**: Refers to actively moving data within a system's pipeline.
2. **Schema Registry**: Ensures structural consistency and schema evolution, enabling seamless interaction between producers and consumers.
3. **Data Contracts**: Bind producers and consumers, guaranteeing both structural and semantic compliance through rulesets.
4. **Stream Governance**: Enforces data quality, compliance, and transformation in real time.

### Governance Strategies
- **"Shift-Left" Approach**: Validates and governs data as early as possible in the pipeline.
- **Dead Letter Queues (DLQ)**: Captures non-compliant messages for future review.
- **Schema Evolution and Migration**: Supports backward, forward, and full compatibility to accommodate changes without breaking existing systems.

---

## Prerequisites

To run the examples in this repository, you need the following installed on your system:

- **Docker**: For deploying the Confluent Platform.
- **curl**: To interact with RESTful APIs.
- **jq**: A command-line tool for JSON manipulation.
- **Confluent Platform**: Download and configure the platform binaries.
- **Visual Studio Code (Optional)**: Recommended for an enhanced development experience.

---

## Getting Started

### Setting Up the Environment

1. Clone this repository and navigate to the root directory.
2. Start the Confluent Platform:
   ```bash
   docker-compose up -d
   ```
3. Access the Confluent Control Center:
   [http://localhost:9021/clusters](http://localhost:9021/clusters)

---

## Examples

### Example 1: Structural Governance
- **Objective**: Validate Avro messages against a predefined schema.
- **Steps**:
  1. Register the `order-transaction` schema using the Schema Registry.
  2. Use `kafka-avro-console-producer` to send compliant messages to a Kafka topic.
  3. Consume and verify the data using `kafka-avro-console-consumer`.

### Example 2: Data Quality Governance
- **Objective**: Add and enforce semantic rules using Confluent Data Contracts.
- **Ruleset**: Ensure the `age` field in messages is greater than 18.
- **Enhancements**:
  - Integrate a **Dead Letter Queue** to handle rejected messages.

### Example 3: Complex Data Quality Governance
- **Objective**: Validate and enforce credit card rules (e.g., length, starting digit).
- **Ruleset**:
  - Enforce field-specific constraints for Visa, Mastercard, and AMEX.

### Example 4: Data Transformation Governance
- **Objective**: Implement schema-breaking changes while maintaining compatibility.
- **Method**:
  - Use **JSONata** migration rules for seamless schema transformation.

---

## How to Run Each Example

Detailed instructions for running each example are provided in the `/examples` directory, including:
- Sample schemas (`.avsc` files)
- Rule sets (`.json` files)
- Scripts for producing and consuming messages

---

## Artifacts

The following artifacts are included:
- **Schemas**: Define the structure of data.
- **Rule Sets**: Specify data quality and governance rules.
- **Docker Compose Config**: Launch the Confluent Platform ecosystem.

---

## Troubleshooting

### Common Issues
1. **Unhealthy Cluster**: Refresh the Control Center's overview page.
2. **Schema Validation Errors**: Ensure your message conforms to the registered schema.

### Tips
- Enable detailed logs for debugging.
- Use the Confluent Control Center to inspect Kafka topics and schemas.

---

## References

- [Confluent Platform Documentation](https://docs.confluent.io/platform/current/overview.html)
- [Docker Installation](https://www.docker.com/products/docker-desktop)
- [jq Documentation](https://jqlang.github.io/jq/)

---

## License

This project is licensed under the MIT License. See the LICENSE file for details.

--- 

