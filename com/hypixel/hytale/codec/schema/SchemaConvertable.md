---
description: Architectural reference for SchemaConvertable
---

# SchemaConvertable

**Package:** com.hypixel.hytale.codec.schema
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface SchemaConvertable<T> {
```

## Architecture & Concepts
The SchemaConvertable interface establishes a fundamental contract within Hytale's data serialization framework. It is the primary mechanism for converting a complex, in-memory Java object into a standardized, data-centric representation known as a Schema. This pattern is central to network replication, persistence (saving games), and configuration management.

Any class that implements SchemaConvertable signals to the engine that its state can be captured and transmitted or stored. It acts as the "serialization" half of a symmetric serialization/deserialization system. The generic type parameter T is used to align the object's type with its schema representation, ensuring type safety and consistency during the conversion process.

The conversion is not a simple one-to-one field mapping. It requires a SchemaContext, implying that the serialization process is context-aware. This context may provide access to shared object registries, versioning information, or other environmental data necessary to correctly build the Schema.

## Lifecycle & Ownership
As an interface, SchemaConvertable does not have a lifecycle of its own. Instead, it imposes lifecycle considerations on the *implementing classes* and the *conversion process*.

-   **Creation:** N/A. The contract exists at compile time. Objects that implement this interface are created according to their own specific lifecycles (e.g., a Player entity is created when a player joins the server).
-   **Scope:** The contract is application-scoped. The conversion operation itself, the call to toSchema, is a transient, short-lived process. It is typically invoked on-demand by a higher-level system like a network manager or world saver.
-   **Destruction:** N/A.

**WARNING:** Implementations of the toSchema method should be idempotent and free of side effects. Calling the method multiple times on an unchanged object should produce an equivalent Schema.

## Internal State & Concurrency
The interface itself is stateless. The responsibility for state management and thread safety lies entirely with the implementing class.

-   **State:** Implementations are expected to read from their own internal state to produce the Schema. The conversion process should not modify the object's state.
-   **Thread Safety:** Implementations **must be thread-safe** if the object can be accessed by multiple threads. For example, an entity's state might be read by a network thread for serialization while being updated by the main game thread. In such cases, the toSchema method must use appropriate concurrency controls (e.g., locks, concurrent collections, or immutable snapshots) to prevent data races and ensure a consistent view of the object's state is serialized.

## API Surface
The public contract is minimal, focusing exclusively on the conversion operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toSchema(SchemaContext) | Schema | O(N) | The core method. Converts the object's state into a Schema representation. Complexity is proportional to the complexity of the object's data. |
| toSchema(SchemaContext, T) | Schema | O(N) | A default method that delegates to the primary toSchema method, ignoring the provided default value. This suggests its counterpart is used in deserialization. |

## Integration Patterns

### Standard Usage
This interface should be implemented by any Plain Old Java Object (POJO) or data-centric game object that represents persistent or transmissible state. A managing system will then invoke the conversion method when needed.

```java
// An object representing a game setting that needs to be saved
public class GameSettings implements SchemaConvertable<GameSettings> {
    private int difficulty;

    @Nonnull
    @Override
    public Schema toSchema(@Nonnull SchemaContext context) {
        // Implementation logic to convert 'difficulty' into a Schema object
        // ...
    }
}

// A system that saves the settings
public void saveConfiguration(GameSettings settings) {
    SchemaContext ctx = ...;
    Schema settingsSchema = settings.toSchema(ctx);
    // write schema to disk...
}
```

### Anti-Patterns (Do NOT do this)
-   **Implementing on Volatile Objects:** Do not implement this interface on objects whose state is purely transient and irrelevant to persistence or networking, such as a UI renderer or a temporary particle effect controller.
-   **Side Effects:** The toSchema method must not modify the object's state. It is a read-only operation. Modifying state during serialization can lead to unpredictable behavior and bugs that are difficult to trace.
-   **Heavy Computation:** Avoid performing computationally expensive operations within toSchema. This method can be called frequently, and blocking a network or main thread will cause severe performance degradation. Pre-calculate or cache data where possible.

## Data Pipeline
SchemaConvertable is a critical entry point into the engine's data serialization pipeline.

> Flow:
> In-Memory Game Object -> `toSchema()` Invocation -> **SchemaConvertable Implementation** -> `Schema` Object -> Schema Codec -> Serialized Byte Stream (for Network/Disk)

