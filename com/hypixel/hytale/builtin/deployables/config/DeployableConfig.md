---
description: Architectural reference for DeployableConfig
---

# DeployableConfig

**Package:** com.hypixel.hytale.builtin.deployables.config
**Type:** Configuration Object

## Definition
```java
// Signature
public abstract class DeployableConfig implements NetworkSerializable<com.hypixel.hytale.protocol.DeployableConfig> {
```

## Architecture & Concepts

The **DeployableConfig** class is an abstract base that serves as a data-driven blueprint for all "deployable" entities within the game world, such as turrets, traps, or other placeable objects. It is not a live entity itself, but rather the static, immutable definition from which live entities are configured and spawned.

Its primary architectural role is to bridge the gap between human-readable configuration files (e.g., JSON assets) and the server's high-performance entity component system (ECS). This is achieved through a powerful and extensible serialization system built upon Hytale's **Codec** library.

The static **CODEC** field, a **CodecMapCodec**, is the cornerstone of this system. It enables polymorphic deserialization by inspecting a "Type" field within the asset file. Based on this type, it selects the appropriate concrete subclass of **DeployableConfig** to instantiate, allowing for a wide variety of deployable behaviors defined entirely in data.

Upon deserialization, the **processConfig** method is invoked as a post-processing step. This critical function transforms human-friendly asset identifiers (e.g., sound event names as Strings) into more efficient, engine-friendly integer indices. This pre-computation optimizes runtime performance by avoiding repeated string lookups during gameplay.

## Lifecycle & Ownership

-   **Creation:** **DeployableConfig** instances are never instantiated directly using the *new* keyword. They are created exclusively by the server's asset loading system at startup. The central **DeployableConfig.CODEC** reads a corresponding data file (e.g., a JSON file) and constructs the object graph in memory.
-   **Scope:** A **DeployableConfig** object is a global, static asset. Once loaded, it persists for the entire lifetime of the server session. It is intended to be read-only after the initial loading and processing phase.
-   **Destruction:** All loaded configurations are destroyed and garbage collected when the server process shuts down.

## Internal State & Concurrency

-   **State:** The state is mutable only during the asset loading and deserialization phase. After the **processConfig** method completes, the object should be treated as **effectively immutable**. Several fields, such as **generatedModel**, are populated via lazy initialization upon first access.

-   **Thread Safety:** This class is **not thread-safe**. The lazy initialization patterns within methods like **getModel** and **getModelPreview** can introduce race conditions if accessed concurrently from multiple threads before their initial invocation.

    **WARNING:** All access to **DeployableConfig** instances must be externally synchronized or confined to a single thread (e.g., the main game loop thread). The engine's design assumes that asset loading and processing are completed before the config is accessed by concurrent gameplay systems.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | void | O(1) | Template method invoked each game tick for a live deployable entity using this config. Intended for subclass override. |
| firstTick(...) | void | O(1) | Template method invoked on the first game tick after a deployable entity is spawned. Intended for subclass override. |
| getModel() | Model | O(1) Amortized | Returns the visual model for the deployable. Lazily initializes the model on first call. **Not thread-safe.** |
| getModelPreview() | Model | O(1) Amortized | Returns the preview model (e.g., for UI). Lazily initializes the model on first call. **Not thread-safe.** |
| toPacket() | DeployableConfig | O(N) | Serializes a subset of the configuration into a network packet for transmission to the client. |

## Integration Patterns

### Standard Usage

Developers do not interact with this class directly. Instead, they define deployables in asset files. The primary extension point is to create a concrete subclass that overrides the **tick** and **firstTick** methods to implement custom game logic.

```java
// Example of a concrete subclass (not shown in provided code)
public class TurretDeployableConfig extends DeployableConfig {
    // Custom fields loaded from JSON...
    private float scanRadius;

    @Override
    public void tick(
        @Nonnull DeployableComponent deployable,
        float dt,
        int index,
        @Nonnull ArchetypeChunk<EntityStore> chunk,
        @Nonnull Store<EntityStore> store,
        @Nonnull CommandBuffer<EntityStore> commands
    ) {
        // Implement turret scanning and firing logic here,
        // using the properties from this config object.
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance of a **DeployableConfig** subclass with *new*. Doing so bypasses the critical **Codec** deserialization and the **processConfig** step, leaving the object in an incomplete and invalid state.
-   **State Mutation:** Do not modify the state of a **DeployableConfig** object after it has been loaded by the server. These objects are shared across the entire server and are expected to be read-only.
-   **Concurrent Access:** Do not access a config instance from multiple threads simultaneously without external locking. The lazy initializers for models are not thread-safe and can lead to unpredictable behavior.

## Data Pipeline

The flow of data from asset file to in-game behavior is a well-defined pipeline.

> Flow:
> JSON Asset File -> Server Asset Loader -> **DeployableConfig.CODEC** (Deserialization) -> Concrete **DeployableConfig** Instance -> **processConfig** (Post-Processing & Indexing) -> Asset Registry -> Deployable Entity System (Spawning & Ticking)

