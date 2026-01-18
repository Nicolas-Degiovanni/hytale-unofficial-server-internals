---
description: Architectural reference for DeployableTrapSpawnerConfig
---

# DeployableTrapSpawnerConfig

**Package:** com.hypixel.hytale.builtin.deployables.config
**Type:** Data-Driven Configuration

## Definition
```java
// Signature
public class DeployableTrapSpawnerConfig extends DeployableTrapConfig {
```

## Architecture & Concepts

The DeployableTrapSpawnerConfig is a specialized configuration class that defines the behavior of a trap-like entity whose primary function is to spawn other deployable entities upon activation. It extends the base DeployableTrapConfig, inheriting its detection and triggering logic, but adds a unique payload: spawning a predefined set of child deployables.

This class is a cornerstone of Hytale's data-driven Entity Component System (ECS). It is not instantiated directly in game logic. Instead, it is deserialized from asset files (e.g., JSON) via its static CODEC field. This allows game designers to create complex spawning behaviors without writing new Java code.

The core logic is implemented as a finite state machine within the **tick** method. An entity associated with this config progresses through a series of states, managed by a flag in its DeployableComponent:

1.  **Deployment (State 0 & 1):** Initial placement and animation.
2.  **Fuze (State 2):** A timed delay before the trap becomes active.
3.  **Live (State 3):** The trap is armed and actively scanning for targets within its radius.
4.  **Triggered (State 4):** The trap has been activated. In this state, it iterates through its list of configured spawners and queues commands to create new deployable entities.
5.  **Despawn (State 5 & 6):** The trap has fulfilled its purpose and begins the cleanup process.

This class acts as a stateless service object. The configuration data it holds is immutable after loading. All per-instance runtime data (e.g., current state, owner, position) is stored in the corresponding DeployableComponent within the ECS.

## Lifecycle & Ownership

-   **Creation:** Instantiated once by the engine's Codec and Asset systems during server startup or when assets are loaded. The static CODEC field defines how to deserialize the configuration from data files into a Java object. The **afterDecode** hook is critical for resolving string-based asset IDs (deployableSpawnerIds) into direct object references (deployableSpawners).

-   **Scope:** This object is an asset. A single instance of DeployableTrapSpawnerConfig exists for each unique spawner trap definition. It persists for the entire server session and is shared by all in-world trap entities of its type (Flyweight pattern).

-   **Destruction:** The object is eligible for garbage collection only when the server's asset registry is unloaded, typically during a full server shutdown.

## Internal State & Concurrency

-   **State:** The object's state is effectively immutable after the initial asset loading and decoding process. Fields like deployableSpawners are populated once by the CODEC's **afterDecode** hook and are not modified during runtime. This design ensures that the configuration remains consistent and predictable.

-   **Thread Safety:** This class is thread-safe for reads due to its immutable state. However, the **tick** method is **not re-entrant** and is fundamentally unsafe to call from any thread other than the world's main update thread. It directly manipulates the ECS state via the Store and CommandBuffer parameters, which are not designed for concurrent access. The engine's scheduler is responsible for ensuring all system ticks occur on the correct thread.

## API Surface

The public API is minimal, designed to be invoked exclusively by the engine's DeployableSystem.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | void | O(1) | Executes one logic step for a single deployable entity. Manages the internal state machine. **WARNING:** Must only be called by the appropriate ECS system on the main world thread. |
| onTriggered(store, ref) | void | O(1) | Overridden callback. Transitions the entity's state to Triggered (4), initiating the spawn sequence on the next tick. |

## Integration Patterns

### Standard Usage

A game developer or designer does not interact with this class directly via Java code. Instead, they define its properties in a data file (e.g., a JSON asset) which is then loaded by the engine. The engine's DeployableSystem is responsible for invoking the tick method.

The following conceptual example illustrates how the system would use this class:

```java
// This code would exist within an engine system, like DeployableSystem.
// It is NOT typical user code.

// For each entity that has a DeployableComponent...
DeployableComponent component = store.getComponent(entityRef, DeployableComponent.getComponentType());
DeployableConfig config = component.getConfig(); // This could be an instance of DeployableTrapSpawnerConfig

// The system dispatches the tick call to the specific config object.
// This is an example of runtime polymorphism.
config.tick(component, dt, index, archetypeChunk, store, commandBuffer);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new DeployableTrapSpawnerConfig()`. The object will be in an uninitialized and invalid state, as the CODEC is responsible for populating its fields from data. This will result in NullPointerExceptions.

-   **State Mutation:** Do not attempt to modify the `deployableSpawners` array or other fields after the object has been loaded. This violates the immutable configuration principle and can lead to unpredictable behavior across all entities using this config.

-   **External Tick Calls:** Do not call the `tick` method from outside the engine's main update loop. Doing so will cause severe concurrency issues, likely leading to data corruption and server crashes.

## Data Pipeline

The primary data flow for this class begins when it is triggered and ends with the creation of new entities.

> Flow:
> Target Entity Enters Radius -> **DeployableTrapSpawnerConfig::tickLiveState** detects target -> **DeployableTrapSpawnerConfig::onTriggered** is called -> DeployableComponent state flag is set to 4 (Triggered) -> On next tick, **DeployableTrapSpawnerConfig::tickTriggeredState** executes -> It iterates `deployableSpawners` array -> **DeployablesUtils::spawnDeployable** is called for each child -> A "create entity" command is written to the **CommandBuffer** -> The ECS processes the CommandBuffer at the end of the frame, creating the new deployable entities in the world.

