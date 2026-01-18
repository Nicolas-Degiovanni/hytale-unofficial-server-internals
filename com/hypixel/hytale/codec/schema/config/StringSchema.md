---
description: Architectural reference for StringSchema
---

# StringSchema

**Package:** com.hypixel.hytale.codec.schema.config
**Type:** Transient

## Definition
```java
// Signature
public class StringSchema extends Schema {
```

## Architecture & Concepts
The StringSchema class is a data model that represents a set of validation rules for a string value. It is a core component of the configuration and data serialization framework, operating within the `hytale.codec` system. Its primary role is to declaratively define the expected format, content, and constraints of a string field when data is deserialized from a source like a JSON or binary file.

This class acts as the in-memory representation of a schema definition. The static `CODEC` field, a `BuilderCodec`, is the engine that translates a serialized representation (e.g., a JSON object with keys like "pattern" and "minLength") into a fully hydrated StringSchema instance. This allows the system to perform robust, schema-driven validation on configuration files, network packets, and other structured data without hard-coding validation logic.

Specialized fields like `hytaleCommonAsset` and `hytaleCosmeticAsset` indicate that this schema is frequently extended to enforce Hytale-specific rules, particularly for validating asset paths and resource identifiers.

## Lifecycle & Ownership
- **Creation:** A StringSchema instance is almost exclusively created by the `StringSchema.CODEC` during the deserialization process. The codec invokes the default constructor and populates the fields based on the source data. Programmatic creation via `new StringSchema()` or the `constant()` static factory is possible for unit testing or dynamic schema generation but is not the primary operational path.
- **Scope:** The lifetime of a StringSchema object is tied to the parent configuration object or schema registry that loaded it. It is a value object, designed to be loaded, used for validation, and then discarded along with its container.
- **Destruction:** The object is managed by the Java garbage collector. There are no native resources or explicit cleanup methods. It is eligible for collection once all references to it and its parent configuration are released.

## Internal State & Concurrency
- **State:** The internal state is **mutable** and consists of a collection of validation constraints (e.g., `pattern`, `minLength`, `maxLength`). The object acts as a container for these rules, which are populated once during deserialization and subsequently read by validation logic.
- **Thread Safety:** This class is **not thread-safe**. All constraint fields are exposed via public setters, making the object's state vulnerable to race conditions if modified and read from different threads concurrently.

> **Warning:** A StringSchema instance should be treated as effectively immutable after its initial population by the codec. Modifying its state while it is being used for validation by other threads will lead to non-deterministic behavior and is a critical anti-pattern.

## API Surface
The public API consists primarily of standard getters and setters for its constraint fields. The following are the most notable symbols.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setPattern(Pattern pattern) | void | O(1) | Sets the regex pattern from a `java.util.regex.Pattern` object. Throws `IllegalArgumentException` if the pattern contains flags, as they are not supported. |
| constant(String c) | static Schema | O(1) | A factory method to create a simple schema that validates a string against a single, constant value. |

## Integration Patterns

### Standard Usage
A StringSchema is typically defined within a larger configuration file and is not interacted with directly. For programmatic construction, the following pattern is used.

```java
// Example of programmatically defining a schema for an asset path
StringSchema assetPathSchema = new StringSchema();
assetPathSchema.setMinLength(5);
assetPathSchema.setPattern(Pattern.compile("^[a-z0-9_:/]+$"));

// This schema object would then be passed to a validator service
// to check user-provided or configuration-loaded strings.
```

### Anti-Patterns (Do NOT do this)
- **Post-Initialization Mutation:** Modifying a StringSchema object after it has been loaded and is in active use by a validator. This breaks the assumption of a static schema and can cause severe validation inconsistencies.
- **Unsupported Pattern Flags:** Passing a `java.util.regex.Pattern` object with flags (e.g., `Pattern.CASE_INSENSITIVE`) to the `setPattern` method. The schema system does not support regex flags and will raise an exception.

## Data Pipeline
StringSchema is a data structure that is produced by the codec pipeline and consumed by the validation pipeline. It represents a static, declarative rule set.

> Flow:
> Configuration File (JSON, etc.) -> `BuilderCodec` Deserializer -> **StringSchema Instance** -> Configuration Validator -> Validation Result (Success/Failure)

