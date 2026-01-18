---
description: Architectural reference for InstanceEntityConfig
---

# InstanceEntityConfig

**Package:** com.hypixel.hytale.builtin.instances.config
**Type:** Component

## Definition
```java
// Signature
public class InstanceEntityConfig implements Component<EntityStore> {
```

## Architecture & Concepts

The InstanceEntityConfig is a data component within Hytale's Entity-Component-System (ECS) architecture. Its sole purpose is to attach instance-specific configuration to an entity, specifically an entity represented by an EntityStore. This component does not contain any logic; it is a pure data container managed by higher-level game systems.

The primary data it manages is the **WorldReturnPoint**, which defines the precise location an entity should be teleported to upon leaving a game instance. This is critical for features like dungeons, minigames, or any system that temporarily moves a player or NPC to a separate world.

A key architectural feature is the separation of the persistent return point from a transient override:
*   **returnPoint:** The primary, canonical return location. This value is serialized and persisted with the entity's data, ensuring it survives server restarts and world saves.
*   **returnPointOverride:** A temporary, runtime-only return location. This field is marked as transient and is **not** serialized. It allows systems to temporarily override an entity's return destination for a single session without permanently altering its saved state.

The static CODEC field integrates this component with the engine's serialization framework, allowing it to be seamlessly saved to and loaded from disk as part of the world data.

## Lifecycle & Ownership

-   **Creation:** InstanceEntityConfig objects must never be instantiated directly using the constructor. They are created and attached to an entity on-demand by game systems via the static `ensureAndGet` factory method. This is typically invoked by an InstanceManager or a world transition system immediately before an entity is moved into an instanced zone.

-   **Scope:** The lifecycle of an InstanceEntityConfig is strictly bound to the entity to which it is attached. It exists only as long as it remains a component of that entity.

-   **Destruction:** The component is destroyed under two conditions:
    1.  When it is explicitly removed from its parent entity via the `removeAndGet` method.
    2.  When the parent entity itself is destroyed. The component is garbage collected along with the entity.

## Internal State & Concurrency

-   **State:** The component's state is mutable. It primarily consists of two WorldReturnPoint object references, `returnPoint` and `returnPointOverride`, which can be modified at any time through their respective setters.

-   **Thread Safety:** This class is **not thread-safe**. It is a simple data object with no internal locking or synchronization mechanisms. It is designed to be accessed and modified exclusively by the thread that owns the entity it is attached to, which is almost always the main server thread for a given world.

    **WARNING:** Unsynchronized access from worker threads or other game threads will lead to severe race conditions, data corruption, and unpredictable behavior. All interactions with this component must be synchronized with the main game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique, engine-wide type identifier for this component. |
| ensureAndGet(Holder) | static InstanceEntityConfig | O(1) amortized | Idempotently retrieves or creates and attaches this component to a given entity holder. |
| removeAndGet(Holder) | static InstanceEntityConfig | O(1) | Removes the component from an entity holder and returns the removed instance. Returns null if not present. |
| getReturnPoint() | WorldReturnPoint | O(1) | Retrieves the primary, persistent return point. |
| setReturnPoint(point) | void | O(1) | Sets the primary, persistent return point. This change will be saved with the entity. |
| getReturnPointOverride() | WorldReturnPoint | O(1) | Retrieves the transient, temporary return point override. |
| setReturnPointOverride(point) | void | O(1) | Sets the transient return point. This change will **not** be saved with the entity. |
| clone() | InstanceEntityConfig | O(N) | Creates a deep copy of the component and its contained WorldReturnPoint objects. |

## Integration Patterns

### Standard Usage

The component should always be retrieved from an entity holder. It is then configured by a system responsible for managing world transitions.

```java
// A system managing a player's entry into a dungeon
// 'playerEntity' is a Holder<EntityStore> representing the player

// 1. Ensure the component exists and get a reference to it.
InstanceEntityConfig config = InstanceEntityConfig.ensureAndGet(playerEntity);

// 2. Configure the persistent return point to be the player's location before entering.
WorldReturnPoint spawnPoint = new WorldReturnPoint(playerEntity.getWorldId(), playerEntity.getPosition());
config.setReturnPoint(spawnPoint);

// 3. Teleport the player to the dungeon.
// ...teleport logic...
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance with `new InstanceEntityConfig()`. This creates an orphaned component that is not attached to any entity and will be ignored by all engine systems. Always use `InstanceEntityConfig.ensureAndGet(holder)`.

-   **State Mismanagement:** Do not use `setReturnPointOverride` for changes that need to be permanent. The override is designed for temporary, in-memory modifications that are expected to be lost on server shutdown or when the state is no longer needed.

-   **Cross-Thread Modification:** Do not access or modify an InstanceEntityConfig from an asynchronous task or worker thread without first queuing the operation to execute on the entity's owner thread (the main world thread).

## Data Pipeline

InstanceEntityConfig acts as a data container within the entity persistence pipeline. It does not process data itself but holds state that is read and written by other systems.

> **Write Flow (Entering an Instance):**
> Game Logic (Instance Manager) -> `ensureAndGet` -> `setReturnPoint` -> **InstanceEntityConfig** (State is held on Entity) -> World Save Event -> Entity Serialization System -> `CODEC` -> Persistent Storage (Disk)

> **Read Flow (Leaving an Instance):**
> Persistent Storage (Disk) -> `CODEC` -> Entity Deserialization System -> **InstanceEntityConfig** (State is restored on Entity) -> Game Logic (Teleport System) -> `getReturnPoint` -> Player Teleportation
---

