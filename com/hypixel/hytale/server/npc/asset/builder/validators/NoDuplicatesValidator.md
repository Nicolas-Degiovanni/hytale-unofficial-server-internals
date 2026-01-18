---
description: Architectural reference for NoDuplicatesValidator
---

# NoDuplicatesValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Transient Utility

## Definition
```java
// Signature
public class NoDuplicatesValidator<T> extends Validator {
```

## Architecture & Concepts

The NoDuplicatesValidator is a specific, single-purpose policy object within the server's asset validation framework. Its primary function is to enforce data integrity by ensuring that a given collection of elements contains no duplicate entries. This component is a concrete implementation of the abstract Validator base class.

Architecturally, this class is designed to be used during the asset construction pipeline, particularly for NPC definitions. It acts as a gatekeeper, preventing the creation of malformed or logically inconsistent NPC assets before they are loaded into the game world. By operating on raw data structures (any object that implements Iterable), it decouples the validation logic from the final, materialized game object, allowing for early failure detection during server bootstrap or asset reloading.

Its design, featuring a private constructor and a static factory method, promotes its use as an immutable, ephemeral tool. A developer does not manage the state of the validator; they simply create it with the data to be validated and immediately execute the check.

## Lifecycle & Ownership

- **Creation:** Instantiation is strictly controlled by the static factory method `withAttributes`. This is typically invoked by a higher-level asset builder or a configuration loader that is parsing an NPC definition file. The private constructor prevents direct instantiation, enforcing the intended creation pattern.
- **Scope:** An instance of NoDuplicatesValidator is extremely short-lived. Its lifecycle is confined to a single validation operation. It is created, its `test` method is invoked, and it immediately becomes eligible for garbage collection.
- **Destruction:** The object holds no native resources or persistent state, so its memory is reclaimed by the Java Garbage Collector when it is no longer referenced. No explicit cleanup is required.

## Internal State & Concurrency

- **State:** The NoDuplicatesValidator is **immutable**. Its internal fields, `iterable` and `variableName`, are marked as final and are set only once during creation via the factory method. The `test` method performs its work by creating a local `HashSet`, which does not alter the instance's state.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable nature, a single instance can be safely shared and used across multiple threads without risk of data corruption or race conditions. All state manipulated during the validation check is local to the `test` method's execution stack.

## API Surface

The public contract is minimal, focusing exclusively on creation and execution of the validation rule.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| withAttributes(Iterable, String) | NoDuplicatesValidator | O(1) | Static factory method. Creates and returns a new validator instance for the given collection. |
| test() | boolean | O(N) | Executes the validation logic. Returns true if all elements are unique, false otherwise. N is the number of elements in the iterable. |
| errorMessage() | String | O(1) | Returns a pre-formatted, human-readable error message if the validation fails. |

## Integration Patterns

### Standard Usage

This validator is intended to be used as part of a larger validation sequence within an asset builder. The builder creates the validator, executes the test, and uses the result to either proceed with asset creation or halt with an error.

```java
// Example within a hypothetical NpcAssetBuilder
List<String> animationNames = config.getAnimationNames();

// Create and execute the validator in a single chain
Validator duplicateCheck = NoDuplicatesValidator.withAttributes(animationNames, "animationNames");
if (!duplicateCheck.test()) {
    throw new AssetCreationException(duplicateCheck.errorMessage());
}

// ... continue building the asset
```

### Anti-Patterns (Do NOT do this)

- **Direct Instantiation:** The constructor is private. Do not attempt to use reflection to create an instance. Always use the `withAttributes` static factory method.
- **Null Inputs:** The `withAttributes` method is annotated with Nonnull. Passing a null `iterable` or `variableName` will result in a NullPointerException at runtime. Always ensure inputs are sanitized before validation.
- **Stateful Misuse:** Do not cache and reuse a validator instance for different collections. The validator is bound to the specific `iterable` it was created with. It is a cheap, transient object that should be created anew for each validation task.

## Data Pipeline

The NoDuplicatesValidator functions as a discrete step in the broader NPC asset loading and validation pipeline. It does not handle I/O but rather operates on in-memory data that has already been parsed.

> Flow:
> NPC Definition File (e.g., JSON) -> Parser -> Raw Data Object (with a List) -> **NoDuplicatesValidator** -> Validation Result (boolean) -> Asset Builder Logic -> Final In-Memory NPC Asset

