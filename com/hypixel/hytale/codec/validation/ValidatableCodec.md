---
description: Architectural reference for ValidatableCodec
---

# ValidatableCodec

**Package:** com.hypixel.hytale.codec.validation
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface ValidatableCodec<T> extends Codec<T> {
```

## Architecture & Concepts
The ValidatableCodec interface extends the core Codec contract to introduce a critical data integrity layer into the serialization and deserialization pipeline. Its primary function is to enforce domain-specific rules on data *after* it has been successfully decoded from its binary or text representation but *before* it is consumed by game systems.

This interface embodies a defensive programming strategy, acting as a gatekeeper to prevent malformed, inconsistent, or logically invalid data from entering the engine's runtime state. Implementations of this interface are responsible for defining what constitutes a "valid" object, which can range from simple null checks and range validation to complex cross-field consistency verification.

The system is designed to be transparently integrated. Core engine components that process data via codecs can check if a given Codec implements ValidatableCodec and, if so, automatically invoke the validation methods, thus ensuring that data integrity checks are consistently applied across the entire application.

## Lifecycle & Ownership
As an interface, ValidatableCodec does not have a lifecycle of its own. The lifecycle described here pertains to its concrete implementations.

- **Creation:** Implementations are typically instantiated as stateless singletons during the engine's bootstrap phase. They are then registered with a central CodecRegistry, making them available for lookup throughout the application.
- **Scope:** An implementation's scope is almost always global, persisting for the entire application session. They are designed to be long-lived and reusable.
- **Destruction:** Instances are destroyed when the containing CodecRegistry is cleared, which typically occurs during application shutdown.

## Internal State & Concurrency
- **State:** Implementations of ValidatableCodec are **required** to be stateless. The validation logic must operate exclusively on the input parameters provided to its methods. Storing state between calls is a severe anti-pattern that will lead to unpredictable behavior in a multithreaded environment.
- **Thread Safety:** All implementations **must be thread-safe**. The codec system is heavily utilized by concurrent systems, including network processing threads, asset loaders, and the main game loop. Validation methods can and will be invoked from multiple threads simultaneously. Any implementation that is not thread-safe will introduce critical race conditions and data corruption bugs.

## API Surface
The API defines the contract for performing validation at two distinct stages: on a decoded object instance and on the codec's default configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| validate(T object, ExtraInfo info) | void | O(N) | Validates a fully decoded object. Throws a runtime exception, typically a ValidationException, if the object fails validation rules. Complexity is dependent on the implementation. |
| validateDefaults(ExtraInfo info, Set<Codec<?>> visited) | void | O(N) | Validates the default state or configuration of the codec itself. This is crucial for detecting configuration errors at startup. The visited set is used to prevent infinite recursion in nested codec structures. |
| validateDefaults(Codec<?> codec, ExtraInfo info, Set<Codec<?>> visited) | static void | O(D) | A static utility for invoking `validateDefaults` on a codec. It correctly unwraps any number of WrappedCodec layers to find the underlying ValidatableCodec to execute. D is the depth of wrapping. |

## Integration Patterns

### Standard Usage
The primary pattern is to implement this interface on a custom Codec. The engine's data processing systems will then automatically invoke the validation logic. Direct invocation is rare.

```java
// A system implementing this interface for a GameSettings object
public class GameSettingsCodec implements ValidatableCodec<GameSettings> {

    // ... implementation of Codec<GameSettings> methods ...

    @Override
    public void validate(GameSettings settings, ExtraInfo info) {
        if (settings.getRenderDistance() < 2) {
            throw new ValidationException("Render distance cannot be less than 2");
        }
        if (settings.getPlayerName() == null || settings.getPlayerName().isEmpty()) {
            throw new ValidationException("Player name cannot be null or empty");
        }
    }

    @Override
    public void validateDefaults(ExtraInfo info, Set<Codec<?>> visited) {
        // Used to validate the default-constructed state if necessary.
        // Often left empty if default state is guaranteed to be valid.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Catching and Ignoring Exceptions:** The validation methods throw exceptions for a critical reason. Swallowing a ValidationException defeats the purpose of the entire system and allows corrupt data to flow into the game state.
- **Stateful Implementations:** Do not add member fields to a ValidatableCodec implementation to store state between calls. This immediately breaks thread safety and will cause unpredictable failures under load.
- **Complex Object Graph Traversal:** Validation logic should be self-contained. Avoid having one validator reach out to other services or traverse large object graphs, as this can lead to poor performance and re-entrancy problems.

## Data Pipeline
This interface inserts a validation gate into the standard data decoding pipeline.

> Flow:
> Raw Data (Network, Disk) -> `Codec.decode` -> Decoded Object -> **`ValidatableCodec.validate`** -> (If valid) -> Game System Consumption
>
> If validation fails, an exception is thrown, halting the pipeline and preventing the invalid object from being used.

