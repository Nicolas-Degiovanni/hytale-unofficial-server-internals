---
description: Architectural reference for PlayerMemories
---

# PlayerMemories

**Package:** com.hypixel.hytale.builtin.adventure.memories.component
**Type:** Data Component

## Definition
```java
// Signature
public class PlayerMemories implements Component<EntityStore> {
```

## Architecture & Concepts
The PlayerMemories class is a data component within Hytale's Entity-Component-System (ECS) architecture. It serves as a state container, or "data bag", that attaches to a player entity to track significant events or discoveries, termed **Memories**. It holds no logic of its own; its data is manipulated by external game systems.

This component is designed for persistence and replication. The static CODEC field provides the necessary serialization and deserialization logic, allowing the engine to save a player's memories to disk as part of the world data and restore them when the player rejoins. This is a fundamental part of the adventure mode gameplay loop, ensuring player progression is not lost between sessions.

Its registration with the engine is managed via the MemoriesPlugin, which provides a unique ComponentType. This allows the core ECS framework to efficiently query for and manage entities that possess this component.

## Lifecycle & Ownership
- **Creation:** PlayerMemories instances are not created directly by developers. They are instantiated by the ECS framework under two conditions:
    1.  A game system explicitly adds the component to an entity, typically a player, for the first time.
    2.  An entity is loaded from storage, and the `EntityStore` deserializes the saved data using the component's CODEC.

- **Scope:** The lifecycle of a PlayerMemories component is strictly bound to the entity it is attached to. It persists as long as the parent entity exists in the world.

- **Destruction:** The component is marked for garbage collection when its parent entity is removed from the world or unloaded. There is no manual destruction method. The `takeMemories` method can be used to clear the component's state without destroying the component itself.

## Internal State & Concurrency
- **State:** The internal state is mutable, consisting of a LinkedHashSet of Memory objects and an integer representing the maximum capacity. The `recordMemory` and `takeMemories` methods directly mutate the internal set.

- **Thread Safety:** **This component is not thread-safe.** All internal collections are non-synchronized. The Hytale game engine guarantees that component data for a given entity is only accessed from a single thread during the main game tick.

    **WARNING:** Any attempt to read or write to a PlayerMemories instance from an asynchronous task or a separate thread will result in race conditions, ConcurrentModificationExceptions, and severe data corruption. All interactions must be scheduled and executed on the main server thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the central plugin registry. |
| clone() | Component | O(N) | Creates a copy of the component, where N is the number of recorded memories. Used internally by the engine. |
| recordMemory(Memory memory) | boolean | O(1) | Attempts to add a new Memory. Fails if the memory already exists or if capacity is full. |
| hasMemories() | boolean | O(1) | Returns true if at least one memory has been recorded. |
| takeMemories(Set outMemories) | boolean | O(N) | Drains all memories into the provided output set and clears the internal collection. This is a destructive read. |
| getRecordedMemories() | Set | O(1) | Returns an **unmodifiable** view of the recorded memories. This is the primary safe method for read-only access. |

## Integration Patterns

### Standard Usage
Interaction with this component should always occur through an entity reference within a game system. The system retrieves the component, performs its logic, and allows the engine to handle persistence.

```java
// Within a server-side game system that has access to a player entity
void grantExplorationMemory(PlayerEntity player, Memory newMemory) {
    // Retrieve the component from the entity. Do not create it.
    PlayerMemories memories = player.getComponent(PlayerMemories.getComponentType());

    if (memories != null) {
        boolean success = memories.recordMemory(newMemory);
        if (success) {
            // Logic to notify player, etc.
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new PlayerMemories()`. A component created this way is untracked by the game engine, will not be persisted, and will not be attached to any entity. It is a memory leak waiting to happen.

- **Concurrent Access:** Do not access a PlayerMemories component from a network callback, a separate thread pool, or any asynchronous task. Schedule all work to be executed on the main game thread.

- **Modifying the Unmodifiable Set:** Attempting to modify the set returned by `getRecordedMemories` will throw an UnsupportedOperationException. This method is for safe, read-only iteration.

## Data Pipeline
PlayerMemories acts as a stateful repository within the server's data flow. It does not actively push data; other systems pull from it or push into it.

> Flow:
> Game Event (e.g., Player enters new biome) -> BiomeDetectionSystem -> **PlayerMemories.recordMemory()** -> State is now held in the component.
>
> Later...
>
> World Save Event -> EntityStore -> **PlayerMemories CODEC (Serialize)** -> Disk Storage
>
> Or...
>
> Player opens UI -> MemoryUISystem -> **PlayerMemories.getRecordedMemories()** -> Data is read and displayed to the user.

