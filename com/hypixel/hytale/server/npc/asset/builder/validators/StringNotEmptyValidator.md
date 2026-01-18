---
description: Architectural reference for StringNotEmptyValidator
---

# StringNotEmptyValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Singleton

## Definition
```java
// Signature
public class StringNotEmptyValidator extends StringValidator {
```

## Architecture & Concepts
The StringNotEmptyValidator is a stateless, reusable component that enforces a single, common validation rule: a string must not be null and must not be empty. It is a concrete implementation of the Strategy pattern, designed to be plugged into a larger validation framework, most notably the NPC asset building pipeline.

Architecturally, this class serves to decouple specific validation logic from the components that require it. By providing a centralized, singleton implementation, the system ensures that the definition of a "non-empty string" is consistent everywhere it is checked. This prevents logic duplication and simplifies maintenance. Its primary role is to act as a predicate in data validation sequences before assets are finalized and loaded into the server.

### Lifecycle & Ownership
-   **Creation:** The single instance of StringNotEmptyValidator is created eagerly by the Java Virtual Machine during static class initialization. The instance is held in the private static final field named INSTANCE.
-   **Scope:** Application-wide. The singleton instance persists for the entire runtime of the server process. It is not tied to any specific session, world, or player.
-   **Destruction:** The object is eligible for garbage collection only when its ClassLoader is unloaded, which typically occurs at server shutdown. There is no manual destruction or cleanup mechanism.

## Internal State & Concurrency
-   **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior depends exclusively on the input arguments provided to its methods. It does not cache any data.
-   **Thread Safety:** This class is inherently thread-safe. As a stateless singleton, it can be safely invoked from multiple threads concurrently without any risk of race conditions or data corruption. No external locking or synchronization is required when using this validator.

## API Surface
The public contract is minimal, focusing entirely on instance retrieval and validation execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static StringNotEmptyValidator | O(1) | Retrieves the global singleton instance of the validator. |
| test(String value) | boolean | O(1) | Executes the validation logic. Returns true if the string is not null and not empty. |
| errorMessage(String value) | String | O(1) | Generates a generic error message for a failed validation. |
| errorMessage(String value, String name) | String | O(1) | Generates a context-specific error message for a failed validation, naming the invalid field. |

## Integration Patterns

### Standard Usage
This validator should always be retrieved via its static get method and used as a predicate within a larger data processing or asset building context.

```java
// Example: Validating an NPC's name during asset construction
String npcNameFromConfig = ...; // Data from a file
StringNotEmptyValidator validator = StringNotEmptyValidator.get();

if (!validator.test(npcNameFromConfig)) {
    throw new AssetBuildException(
        validator.errorMessage(npcNameFromConfig, "NPC Name")
    );
}
// Proceed with asset construction...
```

### Anti-Patterns (Do NOT do this)
-   **Attempted Instantiation:** The constructor is private. Do not attempt to create an instance using reflection. This violates the singleton pattern and provides no benefit.
-   **Redundant Null Checks:** Do not perform a separate null check before calling the test method. The validator's contract explicitly includes checking for null.
    ```java
    // BAD: Redundant and verbose
    if (value != null && StringNotEmptyValidator.get().test(value)) {
        // ...
    }

    // GOOD: Clean and correct
    if (StringNotEmptyValidator.get().test(value)) {
        // ...
    }
    ```

## Data Pipeline
The StringNotEmptyValidator acts as a gate or filter within a larger data transformation pipeline, typically during server bootstrap or asset loading.

> Flow:
> Raw Asset Data (e.g., JSON file) -> Deserializer -> Asset Builder -> **StringNotEmptyValidator** -> Validation Result (Pass/Fail) -> Finalized Server Asset

