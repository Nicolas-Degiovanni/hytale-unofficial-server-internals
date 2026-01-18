---
description: Architectural reference for StringNullOrNotEmptyValidator
---

# StringNullOrNotEmptyValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Singleton

## Definition
```java
// Signature
public class StringNullOrNotEmptyValidator extends StringValidator {
```

## Architecture & Concepts
The StringNullOrNotEmptyValidator is a stateless, reusable component within the server's asset validation framework. Its sole responsibility is to enforce a specific data integrity constraint: a string value must either be null or a non-empty string. This is a common requirement in configuration and asset definitions where a field can be omitted entirely, but if it is present, it must contain meaningful data.

This class implements the Strategy pattern, extending the base StringValidator to provide a concrete validation algorithm. It is designed to be used by higher-level asset builders or configuration parsers during the data ingestion and validation phase, ensuring that NPC assets and related server data conform to expected schemas before being loaded into the game world. Its singleton nature guarantees minimal memory overhead and promotes a consistent, shared validation logic across the entire server application.

### Lifecycle & Ownership
- **Creation:** A single, static instance is eagerly instantiated by the JVM during class loading. This occurs once when the StringNullOrNotEmptyValidator class is first referenced.
- **Scope:** The singleton instance persists for the entire lifetime of the application's classloader. It is globally accessible via the static get method.
- **Destruction:** The instance is eligible for garbage collection only when the application shuts down and its classloader is unloaded. For all practical purposes, it lives forever.

## Internal State & Concurrency
- **State:** This class is **immutable and stateless**. It contains no instance fields and its behavior depends exclusively on the arguments passed to its methods.
- **Thread Safety:** The class is inherently **thread-safe**. As a stateless singleton, it can be safely shared and invoked by multiple threads concurrently without any need for external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String value) | boolean | O(1) | Executes the validation logic. Returns true if the value is null or not empty. |
| errorMessage(String value) | String | O(1) | Generates a generic error message for a failed validation. |
| errorMessage(String value, String name) | String | O(1) | Generates a context-specific error message including the field name. |
| get() | StringNullOrNotEmptyValidator | O(1) | Static factory method to retrieve the singleton instance. |

## Integration Patterns

### Standard Usage
This validator should be retrieved via its static accessor and used within a larger validation process, typically inside an asset builder.

```java
// Example within an asset builder or configuration loader
String potentialName = configuration.getString("npc.displayName");

StringNullOrNotEmptyValidator validator = StringNullOrNotEmptyValidator.get();

if (!validator.test(potentialName)) {
    throw new AssetParseException(validator.errorMessage(potentialName, "displayName"));
}

// If validation passes, proceed
npc.setDisplayName(potentialName);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Attempting to create an instance using reflection violates the singleton design and is strictly forbidden.
- **Redundant Null Checks:** Do not perform a null check before calling the test method. The validator is explicitly designed to handle null as a valid state.

```java
// BAD: Redundant and unnecessary null check
if (value != null && !StringNullOrNotEmptyValidator.get().test(value)) {
    // ...
}

// GOOD: The validator handles the null case correctly
if (!StringNullOrNotEmptyValidator.get().test(value)) {
    // ...
}
```

## Data Pipeline
This component acts as a gate within a larger data processing pipeline, typically for server-side asset loading.

> Flow:
> Raw Asset File (e.g., JSON) -> Deserializer -> Raw Data Object -> Asset Builder -> **StringNullOrNotEmptyValidator** -> Validated Game Asset

