---
description: Architectural reference for MinecartComponent
---

# MinecartComponent

**Package:** com.hypixel.hytale.builtin.mounts.minecart
**Type:** Data Component (Transient)

## Definition
```java
// Signature
public class MinecartComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The MinecartComponent is a fundamental data-holding class within Hytale's Entity-Component-System (ECS) architecture. It does not contain any logic. Its sole responsibility is to store the state for an entity that behaves as a minecart.

This component is designed to be attached to an Entity to grant it minecart-specific properties, such as its durability (number of hits) and the item used to create it.

A critical feature is the static CODEC field. This enables the engine's serialization framework to save and load the component's state to and from persistent storage (via the EntityStore) or transmit it over the network. This mechanism is essential for world persistence and multiplayer synchronization. The component's type is registered with the engine via the MountPlugin, ensuring it can be dynamically looked up and managed by game systems.

## Lifecycle & Ownership
- **Creation:** A MinecartComponent is never instantiated directly by game logic. It is created and attached to an Entity by a higher-level system, typically when a player places a minecart item in the world. The private constructor is reserved for use by the serialization CODEC during deserialization from storage.
- **Scope:** The lifecycle of a MinecartComponent instance is strictly bound to the lifecycle of its parent Entity. It exists only as long as the Entity exists in the world.
- **Destruction:** The component is marked for garbage collection when its parent Entity is destroyed or when the component is explicitly removed from the Entity. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The internal state is fully mutable. Fields such as numberOfHits and lastHit are expected to be frequently updated by game systems (e.g., a DamageSystem or PhysicsSystem) during the game loop.
- **Thread Safety:** This class is **not thread-safe**. As a plain data component in an ECS, it is designed to be accessed and modified exclusively by systems running on the main game thread. Unsynchronized access from other threads will lead to race conditions, data corruption, and unpredictable behavior.

**WARNING:** All interactions with a MinecartComponent instance must be synchronized with the main game loop or managed by a system that guarantees thread-safe access.

## API Surface
The public API is composed of simple data accessors and the core component interface methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | ComponentType | O(1) | Statically retrieves the unique type identifier for this component class from the MountPlugin registry. |
| setNumberOfHits(int) | void | O(1) | Updates the damage counter for the minecart. |
| getLastHit() | Instant | O(1) | Retrieves the timestamp of the last damaging hit. |
| clone() | Component | O(1) | Creates a new instance of the component with a copied sourceItem. Used for entity duplication or blueprinting. |

## Integration Patterns

### Standard Usage
A MinecartComponent should always be retrieved from its parent Entity. Systems that operate on minecarts will query for entities that possess this component and then retrieve it to read or modify its state.

```java
// A hypothetical system processing a minecart entity
void processMinecart(Entity entity) {
    // Retrieve the component from the entity if it exists
    MinecartComponent minecart = entity.getComponent(MinecartComponent.getComponentType());

    if (minecart != null) {
        // Read state
        int currentHits = minecart.getNumberOfHits();

        // Modify state
        minecart.setNumberOfHits(currentHits + 1);
        minecart.setLastHit(Instant.now());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new MinecartComponent()`. This creates an orphan component that is not managed by any Entity or the game engine, leading to memory leaks and systems being unable to find it.
- **Unmanaged State:** Do not store a long-term reference to a MinecartComponent. Its validity is tied to its parent Entity. If the Entity is destroyed, a stored reference becomes stale and will cause NullPointerExceptions or use-after-free bugs.
- **Cross-Thread Modification:** Do not access or modify a component from a worker thread without explicit synchronization through the engine's task scheduler. This will corrupt game state.

## Data Pipeline
The MinecartComponent acts as a data source and sink at the entity level. Its primary data flow is between game systems and the engine's persistence layer.

> **Serialization (World Save):**
> Game Systems -> **MinecartComponent** (State Update) -> EntityStore -> CODEC Serialization -> Persistent Storage (Disk)

> **Deserialization (World Load):**
> Persistent Storage (Disk) -> CODEC Deserialization -> **MinecartComponent** (Instantiation) -> Entity Attachment -> Game Systems

