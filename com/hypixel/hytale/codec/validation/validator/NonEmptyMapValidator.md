---
description: Architectural reference for NonEmptyMapValidator
---

# NonEmptyMapValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Singleton / Utility

## Definition
```java
// Signature
public class NonEmptyMapValidator<K, V> extends NonNullValidator<Map<K, V>> {
```

## Architecture & Concepts
The NonEmptyMapValidator is a foundational component within the data validation framework, operating primarily within the codec and data model layers. Its purpose is to enforce a simple but critical constraint: that a given Map data structure must contain at least one entry.

Architecturally, it is implemented as a stateless, reusable singleton. This design choice is deliberate to eliminate object allocation overhead during high-frequency validation scenarios, such as deserializing network payloads or processing configuration files. By providing a single, globally accessible INSTANCE, the engine avoids the performance cost of repeatedly creating new validator objects.

It extends NonNullValidator, but its own logic (`t == null || t.isEmpty()`) fully encapsulates and expands upon the parent's behavior, making it a more specific and stringent rule.

### Lifecycle & Ownership
- **Creation:** The singleton INSTANCE is instantiated by the Java Virtual Machine's class loader when the NonEmptyMapValidator class is first referenced. Its creation is not managed by any application-level factory or dependency injection framework.
- **Scope:** As a static final field, the INSTANCE has a global scope and persists for the entire lifetime of the application.
- **Destruction:** The object is eligible for garbage collection only when the application's class loader is unloaded, which typically occurs at JVM shutdown. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The NonEmptyMapValidator is **stateless and immutable**. It contains no instance fields and its behavior depends solely on the arguments provided to its methods.
- **Thread Safety:** This class is **unconditionally thread-safe**. Its stateless nature ensures that the singleton INSTANCE can be safely shared and invoked by multiple threads concurrently without any risk of race conditions or need for external synchronization. This is a critical property for validators used in a multi-threaded engine.

## API Surface
The public contract consists of a single operational method inherited and implemented from its validation lineage.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(Map, ValidationResults) | void | O(1) | Assesses if the input Map is null or empty. If either condition is true, it registers a failure in the provided ValidationResults object. |

## Integration Patterns

### Standard Usage
The validator is designed to be used via its public static INSTANCE field. It is typically invoked as part of a larger sequence of validation rules applied to a data object.

```java
// A data object's validation routine
ValidationResults results = new ValidationResults();
Map<String, Entity> entityMap = myGameObject.getEntities();

// Correctly use the globally available singleton instance
NonEmptyMapValidator.INSTANCE.accept(entityMap, results);

if (results.hasFailed()) {
    // Abort processing or log the error
    throw new ValidationException("Entity map cannot be empty.");
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private to enforce the singleton pattern. Do not attempt to bypass this using reflection. Always use NonEmptyMapValidator.INSTANCE.
- **Ignoring Results:** Calling the `accept` method without subsequently checking the state of the ValidationResults object is a logical error. The validation has no effect if its outcome is ignored.
- **Pre-emptive Null Checks:** Writing code that checks if the map is null before passing it to the validator is redundant. The validator is explicitly designed to handle the null case as a failure condition.

## Data Pipeline
The NonEmptyMapValidator acts as a gate in a data processing pipeline. It does not transform data; it asserts conditions on it. A failure reported by the validator should halt the pipeline or divert the data to an error-handling flow.

> Flow:
> Data Source (e.g., Network Packet) -> Deserializer -> Data Transfer Object -> **NonEmptyMapValidator** -> ValidationResults Check -> Business Logic

