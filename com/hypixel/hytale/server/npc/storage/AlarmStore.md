---
description: Architectural reference for AlarmStore
---

# AlarmStore

**Package:** com.hypixel.hytale.server.npc.storage
**Type:** Transient Data Component

## Definition
```java
// Signature
public class AlarmStore extends ParameterStore<Alarm> {
```

## Architecture & Concepts
The AlarmStore is a specialized data container designed to manage a collection of Alarm objects for a server-side entity, typically an NPC. It functions as a component within a larger entity data model, encapsulating state related to timed events or AI behavior triggers.

This class does not contain any logic itself; it is a pure data structure. Its primary architectural significance lies in its integration with Hytale's serialization framework via the static CODEC field. By extending the generic ParameterStore, it inherits the underlying mechanism for managing a map of named parameters, but provides a type-safe specialization for Alarms. This pattern allows game logic systems, such as an AI behavior system, to retrieve and manipulate an entity's alarms without direct knowledge of the underlying storage implementation.

The presence of a `BuilderCodec` is a critical design choice, indicating that the entire state of an AlarmStore is expected to be fully serializable and deserializable. This is essential for game state persistence (saving and loading) and for replicating entity state across the network.

## Lifecycle & Ownership
- **Creation:** An AlarmStore is instantiated when its parent entity is created or deserialized. It is not managed by a global service or registry. For example, when an NPC is loaded from world data, the `CODEC` is used to construct a new AlarmStore instance and populate it with the saved alarm data.
- **Scope:** The lifetime of an AlarmStore is strictly bound to its owning entity. It persists as long as the NPC or other game object exists in the server world.
- **Destruction:** The object is eligible for garbage collection when its owning entity is unloaded or destroyed. There are no explicit cleanup or `destroy` methods.

## Internal State & Concurrency
- **State:** The state of AlarmStore is highly **mutable**. It internally manages a `HashMap` of Alarm objects (inherited from ParameterStore), which can be added, modified, or removed at runtime to reflect changes in the entity's behavior.
- **Thread Safety:** This class is **not thread-safe**. The underlying data structure is a standard `HashMap`, and no synchronization mechanisms are employed. All interactions with an AlarmStore instance must be performed on the main server thread to prevent race conditions, `ConcurrentModificationException`, and other threading-related defects.

## API Surface
The public contract of AlarmStore is primarily defined by its parent, ParameterStore. The most significant symbol defined directly in this class is its serialization codec.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | N/A | **Critical.** Static serializer/deserializer for this class. Used by the engine for persistence and networking. |
| createParameter() | Alarm | O(1) | *Protected.* Factory method used by the parent ParameterStore to instantiate new Alarm objects when requested. |

## Integration Patterns

### Standard Usage
An AlarmStore should be accessed as a component of an entity. Game systems retrieve the store from the entity to schedule or query alarms that drive behavior.

```java
// Example: An AI system retrieving an NPC's AlarmStore
Npc npc = world.getNpcById(npcId);
AlarmStore alarms = npc.getState().get(AlarmStore.class);

// Set an alarm to trigger in 100 ticks (5 seconds)
Alarm attackCooldown = alarms.getOrCreate("attack_cooldown");
attackCooldown.set(100);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid creating instances with `new AlarmStore()`. The object should be managed as part of a larger entity's state component map, ensuring it is correctly initialized and serialized with its owner.
- **Cross-Thread Modification:** Never modify an AlarmStore from an asynchronous task or worker thread. All reads and writes must be synchronized with the main server game loop. Failure to do so will result in unpredictable state corruption.

## Data Pipeline
The AlarmStore serves as a container in the entity data serialization pipeline. It does not process data itself but holds it for processing by other systems.

> **Serialization Flow (Game Save):**
> NpcEntity -> NpcState -> **AlarmStore** -> AlarmStore.CODEC -> Serialized Binary/Text Data -> World Save File

> **Deserialization Flow (Game Load):**
> World Save File -> Serialized Binary/Text Data -> AlarmStore.CODEC -> **AlarmStore** -> NpcState -> NpcEntity

