---
description: Architectural reference for Repulsion
---

# Repulsion

**Package:** com.hypixel.hytale.server.core.modules.entity.repulsion
**Type:** Data Component

## Definition
```java
// Signature
public class Repulsion implements Component<EntityStore> {
```

## Architecture & Concepts
The Repulsion component is a server-side, data-only structure that attaches physical repulsion properties to an entity. Within Hytale's Entity-Component-System (ECS) architecture, this class serves as a plain data container with no inherent logic. Its primary responsibility is to link an entity to a specific set of repulsion behaviors defined in a RepulsionConfig asset.

To optimize for performance and memory, the component does not store the full RepulsionConfig object. Instead, it holds an integer, **repulsionConfigIndex**, which acts as a direct index into a globally available asset map. This pattern of using integer indices instead of string identifiers or direct object references is a common engine optimization for fast lookups and compact data serialization.

The presence of a static CODEC field indicates that this component is fully integrated with the engine's serialization and persistence layer. It can be saved to disk as part of the world's EntityStore and replicated over the network to clients. The component also includes a dirty flag, **isNetworkOutdated**, which is fundamental to the engine's network synchronization strategy, ensuring that changes are efficiently propagated to clients only when necessary.

## Lifecycle & Ownership
- **Creation:** A Repulsion component is instantiated and attached to an entity when that entity is first created with repulsion physics. This is typically handled by a higher-level factory or system, such as an EntityFactory, which constructs entities from predefined archetypes. The primary constructor requires a RepulsionConfig, from which it derives the necessary index.

- **Scope:** The lifecycle of a Repulsion instance is strictly bound to the lifecycle of its parent entity. It persists as long as the entity exists within the server's world simulation.

- **Destruction:** The component is marked for garbage collection when its parent entity is removed from the world or when the component is explicitly detached from the entity. There is no manual destruction method.

## Internal State & Concurrency
- **State:** The component's state is mutable. Its core data, the repulsionConfigIndex, can be changed during runtime to alter an entity's physical behavior dynamically. It also maintains the transient boolean state isNetworkOutdated for network replication purposes.

- **Thread Safety:** **This class is not thread-safe.** As with most ECS components in the engine, it is designed to be accessed and mutated exclusively by the main server game loop thread. Unsynchronized access from other threads will lead to severe race conditions and state corruption, particularly with the non-atomic consumeNetworkOutdated flag. All modifications must be scheduled as tasks to be executed on the main thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally unique type identifier for the Repulsion component. |
| getRepulsionConfigIndex() | int | O(1) | Returns the integer index of the associated RepulsionConfig asset. |
| setRepulsionConfigIndex(int) | void | O(1) | Sets the configuration index. **Warning:** This method does not automatically mark the component as dirty for network updates. |
| consumeNetworkOutdated() | boolean | O(1) | Returns true if the component state has changed and atomically resets the flag to false. This is the primary mechanism for the network system to detect changes. |
| clone() | Component | O(1) | Creates a new Repulsion instance with an identical data state. Used for entity duplication and blueprinting. |

## Integration Patterns

### Standard Usage
The Repulsion component is typically retrieved from an entity to allow a system, such as the physics engine, to look up the full configuration and apply the relevant forces.

```java
// In a server-side system, 'entity' is the target entity.
Repulsion repulsion = entity.getComponent(Repulsion.getComponentType());

if (repulsion != null) {
    int configIndex = repulsion.getRepulsionConfigIndex();
    
    // Use the index to retrieve the full configuration from the asset map.
    RepulsionConfig config = RepulsionConfig.getAssetMap().get(configIndex);
    
    // Apply physics logic based on the properties in 'config'.
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new Repulsion()`. The protected default constructor is reserved for the serialization system (CODEC) and the clone method. Components must be added to entities via the appropriate entity management APIs.

- **State Mutation without Flagging:** Directly calling setRepulsionConfigIndex without informing the broader system to set the network dirty flag will cause state desynchronization between the server and clients. The client's view of the entity will become stale.

- **Multi-threaded Access:** Reading or writing to a Repulsion component from any thread other than the main server thread is strictly forbidden and will lead to unpredictable behavior and crashes.

## Data Pipeline
The data within a Repulsion component primarily flows from game logic to the network and persistence layers. It does not process data itself; it is a container for data that is processed by other systems.

> Flow (State Change):
> Game System (e.g., Skill System) -> Modifies **Repulsion** component on an Entity -> Network Synchronization System (detects change via consumeNetworkOutdated) -> **Repulsion.CODEC** serializes the component state -> Network Packet -> Client deserializes packet and updates its local entity.

