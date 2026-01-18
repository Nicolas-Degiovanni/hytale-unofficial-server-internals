---
description: Architectural reference for StringArrayNoEmptyStringsValidator
---

# StringArrayNoEmptyStringsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Singleton

## Definition
```java
// Signature
public class StringArrayNoEmptyStringsValidator extends StringArrayValidator {
```

## Architecture & Concepts
The StringArrayNoEmptyStringsValidator is a stateless, single-purpose component within the server's asset validation framework. It exists to enforce a specific data integrity rule: a given array of strings must not contain any null or empty string elements.

Architecturally, this class embodies the **Strategy Pattern**. It provides a concrete implementation of a validation algorithm, inheriting from the abstract StringArrayValidator. This design allows higher-level systems, such as an asset builder or configuration loader, to compose different validation rules without being coupled to their specific implementations. Its primary application is during the NPC asset loading pipeline, where it acts as a guard to prevent malformed data from propagating into the game engine.

As a singleton, it ensures that only one instance of this stateless validator exists throughout the application's lifecycle, minimizing memory overhead.

### Lifecycle & Ownership
- **Creation:** The single static instance, INSTANCE, is instantiated eagerly by the JVM during class loading. It is not created by any specific manager or factory at runtime.
- **Scope:** Application-wide. The instance persists for the entire lifetime of the server process.
- **Destruction:** The instance is eligible for garbage collection only when its classloader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior depends solely on the inputs provided to its methods.
- **Thread Safety:** The class is inherently **thread-safe**. Due to its stateless nature, multiple threads can invoke its methods concurrently without any risk of interference or data corruption. No synchronization mechanisms are required.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String[] list) | boolean | O(N) | Evaluates the array. Returns false if any element is null or empty. Returns true if the array is null or valid. |
| errorMessage(String name, String[] list) | String | O(1) | Generates a formatted, context-aware error message for logging or user feedback. |
| get() | StringArrayNoEmptyStringsValidator | O(1) | Static factory method to access the singleton instance. |

## Integration Patterns

### Standard Usage
This validator is intended to be used by asset builders or data processors that need to enforce data quality. The singleton instance should always be retrieved via the static get method.

```java
// Used within a larger asset validation system
String[] textureAliases = assetData.getTextureAliases();

if (!StringArrayNoEmptyStringsValidator.get().test(textureAliases)) {
    throw new AssetValidationException(
        StringArrayNoEmptyStringsValidator.get().errorMessage("textureAliases", textureAliases)
    );
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private to enforce the singleton pattern. Attempting to create an instance using reflection will break the design contract and is strictly forbidden.
- **Redundant Null Checks:** The test method is designed to correctly handle a null input array by returning true. Pre-checking for null before calling the method is unnecessary.
    ```java
    // BAD: Redundant check
    if (myArray != null && !validator.test(myArray)) {
        // ...
    }

    // GOOD: The validator handles the null case
    if (!validator.test(myArray)) {
        // ...
    }
    ```

## Data Pipeline
This validator acts as a conditional gate within a larger data processing pipeline, typically during server startup or asset hot-reloading. It intercepts raw data structures after deserialization to ensure their integrity before they are transformed into engine-ready objects.

> Flow:
> NPC Definition File (JSON) -> Deserializer -> Raw String Array -> **StringArrayNoEmptyStringsValidator** -> (On Failure: Halt & Log) -> (On Success: Continue Processing) -> NPC Asset Object

