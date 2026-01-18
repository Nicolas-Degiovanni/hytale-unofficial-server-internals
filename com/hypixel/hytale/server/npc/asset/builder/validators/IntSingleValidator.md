---
description: Architectural reference for IntSingleValidator
---

# IntSingleValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Flyweight Utility

## Definition
```java
// Signature
public class IntSingleValidator extends IntValidator {
```

## Architecture & Concepts
The IntSingleValidator is a specialized, high-performance component within the server's asset validation framework. Its primary function is to enforce a single relational constraint (e.g., greater than, less than or equal to) on an integer value. It is designed to be used during the parsing and construction of game assets, particularly for validating fields in NPC definition files.

Architecturally, this class implements the **Flyweight Pattern**. It avoids the overhead of object creation by providing pre-configured, static, and immutable instances for common validation scenarios, such as ensuring a value is non-negative. The private constructor is a deliberate design choice to enforce this pattern, preventing external instantiation and forcing consumers to use the provided static factories. This makes it an extremely memory-efficient and fast solution for repetitive validation tasks that occur during server startup or asset hot-reloading.

## Lifecycle & Ownership
- **Creation:** The common validator instances, such as VALIDATOR_GREATER_EQUAL_0, are instantiated once by the JVM during static class initialization. No further instances can be created at runtime due to the private constructor.
- **Scope:** Application-wide. The static instances persist for the entire lifetime of the server process. They are effectively global, stateless services.
- **Destruction:** The objects are garbage collected by the JVM only when the class loader is unloaded, which typically happens during a full application shutdown. No manual cleanup is required or possible.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state, consisting of the RelationalOperator and the integer value to compare against, is set once in the constructor and stored in final fields. The object's state cannot be modified after its creation.
- **Thread Safety:** **Guaranteed thread-safe**. Its immutable nature ensures that a single instance can be safely shared and executed by multiple threads concurrently without any risk of race conditions or data corruption. This is a critical property for the server's multi-threaded asset loading pipeline.

## API Surface
The public contract is minimal, focusing on retrieving pre-built validators and executing the validation logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(int value) | boolean | O(1) | Executes the core validation logic. Returns true if the input value satisfies the validator's relation. |
| errorMessage(int value, String name) | String | O(1) | Generates a detailed, human-readable error message for a failed validation. |
| greaterEqual0() | IntValidator | O(1) | Static factory. Returns a shared instance that validates if a value is greater than or equal to 0. |
| greater0() | IntValidator | O(1) | Static factory. Returns a shared instance that validates if a value is strictly greater than 0. |

## Integration Patterns

### Standard Usage
This class should be used to fetch a validator instance and apply it within a larger data parsing or object building process.

```java
// In an NPC asset builder, retrieve a shared validator instance
IntValidator nonNegativeValidator = IntSingleValidator.greaterEqual0();

// Read a value from a data source
int npcHealth = npcDataSource.getHealth();

// Apply the validation rule
if (!nonNegativeValidator.test(npcHealth)) {
    // Throw a specific exception to halt the asset loading process
    throw new AssetValidationException(
        nonNegativeValidator.errorMessage(npcHealth, "NPC Health")
    );
}
```

### Anti-Patterns (Do NOT do this)
- **Attempting Instantiation:** Do not attempt to create an instance using reflection. The class is designed to be used only through its static factory methods to maintain the Flyweight pattern.
- **Redundant Checks:** Do not fetch the validator inside a loop. Retrieve the required validator once and reuse the instance for all subsequent checks to maximize performance.

## Data Pipeline
The IntSingleValidator acts as a gatekeeper within the broader asset deserialization and validation pipeline. It does not transform data but rather asserts its correctness.

> Flow:
> NPC Asset File (e.g., JSON) -> Jackson Deserializer -> Raw Asset POJO -> Asset Builder -> **IntSingleValidator.test()** -> Validation Result (Pass/Fail)

