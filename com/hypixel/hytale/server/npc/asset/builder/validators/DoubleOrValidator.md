---
description: Architectural reference for DoubleOrValidator
---

# DoubleOrValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility / Immutable Value Object

## Definition
```java
// Signature
public class DoubleOrValidator extends DoubleValidator {
```

## Architecture & Concepts
The DoubleOrValidator is a specialized component within the server-side NPC asset validation framework. Its primary function is to enforce a compound conditional rule on a numeric configuration value, where the value is considered valid if it satisfies one of two distinct relational comparisons.

This class embodies the Strategy pattern, encapsulating a specific validation algorithm (`(value R1 v1) || (value R2 v2)`). It is designed to be used by higher-level asset builders and parsers during the asset loading and verification process. By providing pre-configured static instances for common use cases, such as `greaterEqual0OrMinus1`, it promotes reuse and reduces the likelihood of logic errors in asset definitions.

Its immutability is a core design principle, ensuring that a validator's behavior is consistent and predictable throughout the application lifecycle, which is critical in a multithreaded server environment.

### Lifecycle & Ownership
- **Creation:** The primary intended instance, `GREATER_EQUAL_0_OR_MINUS_1`, is a static final field instantiated once by the JVM during class loading. Direct instantiation by consumers is prohibited via a private constructor. All access is mediated through the static factory method `greaterEqual0OrMinus1`.
- **Scope:** The static instance is application-scoped, persisting for the entire runtime of the server.
- **Destruction:** The object is garbage collected by the JVM when the application terminates and its class loader is unloaded. No manual cleanup is required or possible.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields defining the validation logic (`relationOne`, `valueOne`, `relationTwo`, `valueTwo`) are final and set exclusively at construction time. An instance of DoubleOrValidator cannot be modified after it is created.
- **Thread Safety:** **Fully thread-safe**. Due to its immutable nature, a DoubleOrValidator instance can be safely shared and invoked from any number of concurrent threads without external synchronization. The `test` method is a pure function, producing no side effects.

## API Surface
The public contract is minimal, focusing exclusively on validation and error reporting.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| greaterEqual0OrMinus1() | static DoubleOrValidator | O(1) | Factory method. Returns a shared, static instance that validates if a value is >= 0.0 or == -1.0. |
| test(double value) | boolean | O(1) | Executes the core validation logic. Returns true if the value satisfies either of the two configured conditions. |
| errorMessage(double value, String name) | String | O(1) | Generates a detailed, human-readable error message if validation fails. |

## Integration Patterns

### Standard Usage
The validator should be retrieved from its static factory and used as a predicate within a larger validation or building process.

```java
// Example: Validating an NPC's "cooldown" property during asset parsing.
DoubleOrValidator cooldownValidator = DoubleOrValidator.greaterEqual0OrMinus1();
double cooldownValue = npcConfig.getCooldown(); // Assume this is -1.0

if (!cooldownValidator.test(cooldownValue)) {
    // This block will not be reached for a value of -1.0
    throw new AssetParseException(
        cooldownValidator.errorMessage(cooldownValue, "cooldown")
    );
}

// Cooldown is valid, continue processing...
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create instances using reflection. The class is designed to be used through its provided static factories to ensure use of the intended, shared instances.
- **Re-implementing Logic:** Do not manually check the conditions before calling the validator. The purpose of the class is to encapsulate this logic.

   ```java
   // BAD: Redundant and error-prone
   if ((value >= 0 || value == -1) && validator.test(value)) {
       // ...
   }

   // GOOD: Trust the validator
   if (validator.test(value)) {
       // ...
   }
   ```

## Data Pipeline
DoubleOrValidator acts as a gate within the server's asset ingestion pipeline. It does not process a stream of data but rather validates a single data point at a specific stage.

> Flow:
> NPC Asset File (e.g., JSON) -> Deserialization Engine -> Raw Asset Data Object -> Asset Builder -> **DoubleOrValidator.test()** -> Validated Game Asset

