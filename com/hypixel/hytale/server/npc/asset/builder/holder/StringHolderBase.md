---
description: Architectural reference for StringHolderBase
---

# StringHolderBase

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Component Base Class

## Definition
```java
// Signature
public abstract class StringHolderBase extends ValueHolder {
```

## Architecture & Concepts
StringHolderBase is an abstract foundational component within the server-side NPC asset building framework. It serves as a specialized container for string values that require validation against a broader game state or context. Its primary role is to decouple the data (the string) from the logic that validates it.

This class is a key enabler of **relational integrity** within NPC asset definitions. For example, an NPC asset might define a string field for a `factionId`. A concrete implementation of StringHolderBase would hold this ID, and validators would be attached to ensure that a faction with that specific ID actually exists in the game data.

The system uses an injectable validation model, where `BiConsumer` functions are added as "relation validators". These validators are executed during a dedicated validation phase of the asset loading pipeline, receiving an `ExecutionContext` which provides the necessary context (like access to game registries) to perform their checks. This is an implementation of the **Strategy Pattern**, allowing validation logic to be defined externally and applied dynamically.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses are instantiated by a higher-level asset parser or builder when it encounters a string field in an NPC definition file that requires relational validation.
- **Scope:** The instance's lifetime is tied directly to the in-memory representation of the NPC asset being constructed. It persists as long as the asset definition is being processed.
- **Destruction:** The object is marked for garbage collection when the parent asset object is discarded, typically after the NPC has been fully loaded and its definition is no longer needed by the builder system.

## Internal State & Concurrency
- **State:** The internal state is mutable. It contains a `relationValidators` list which is lazily initialized to conserve memory for fields that do not require validation.
- **Thread Safety:** This class is **not thread-safe**. The `addRelationValidator` method contains a check-then-act race condition on the `relationValidators` field. Furthermore, the underlying `ObjectArrayList` is not a concurrent collection.

   **WARNING:** All operations on instances of this class and its subclasses must be performed on a single thread. The NPC asset building pipeline is designed to be a single-threaded process. Concurrent modification will lead to unpredictable behavior, including `NullPointerException` or lost validators.

## API Surface
The public contract is focused on configuring the validation behavior.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addRelationValidator(validator) | void | O(1) | Adds a validation rule to be executed later. This is the primary configuration method. |
| validateRelations(context, value) | void | O(N) | *Protected.* Executes all registered validators against the provided value and context. N is the number of validators. |

## Integration Patterns

### Standard Usage
A concrete subclass is responsible for holding the actual string value and invoking the validation logic at the correct time. The builder system configures the instance with the necessary validation rules.

```java
// Conceptual example within a hypothetical builder
// MyStringHolder extends StringHolderBase

MyStringHolder factionIdHolder = new MyStringHolder();

// The builder adds a validator that checks if the faction exists
factionIdHolder.addRelationValidator((context, factionId) -> {
    if (!context.getFactionRegistry().exists(factionId)) {
        throw new AssetValidationException("Faction not found: " + factionId);
    }
});

// Later, during a validation pass, the subclass would call:
// factionIdHolder.validate(executionContext);
// which internally calls validateRelations(executionContext, "some_faction_id");
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Access:** Do not share an instance of a StringHolderBase subclass across multiple threads. Do not call `addRelationValidator` from one thread while another might be reading or writing to it.
- **Misusing Protected Methods:** Do not call `validateRelations` from outside the class hierarchy. This method is intended to be invoked by the concrete subclass as part of its own internal validation lifecycle.

## Data Pipeline
This component acts as a validation stage within the broader NPC asset loading pipeline. It does not transform data but rather asserts its correctness.

> Flow:
> NPC Asset File (JSON/HOCON) -> Asset Parser -> **Concrete StringHolder Instantiation** -> Validator Registration -> Asset Validation Phase -> **validateRelations() Execution** -> Throws Exception on Failure or Continues Loading

