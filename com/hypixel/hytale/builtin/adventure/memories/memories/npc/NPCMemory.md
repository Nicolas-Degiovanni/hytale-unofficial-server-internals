---
description: Architectural reference for NPCMemory
---

# NPCMemory

**Package:** com.hypixel.hytale.builtin.adventure.memories.memories.npc
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class NPCMemory extends Memory {
```

## Architecture & Concepts
NPCMemory is a data-centric class that represents a player's discovery of a specific Non-Player Character (NPC) type within the "Memories" gameplay system. It serves as a serializable record, capturing the essential details of the encounter: which NPC was found, when, and where.

Architecturally, this class is a concrete implementation of the abstract Memory concept. Its primary role is to be stored within a player's PlayerMemories component, forming a persistent collection of their discoveries. The class is tightly coupled with its serialization format, defined by a static BuilderCodec instance named CODEC. This mechanism is fundamental to the engine's data persistence and networking strategy, allowing NPCMemory instances to be saved to disk or synchronized with clients.

The uniqueness of an NPCMemory is determined by the NPC's role, accessible via the getId method. This allows the system to prevent duplicate entries for the same NPC type in a player's collection.

## Lifecycle & Ownership
-   **Creation:** NPCMemory instances are created under two conditions:
    1.  **Dynamic Discovery:** Instantiated by the GatherMemoriesSystem when a player enters the proximity of a memory-eligible NPC for the first time.
    2.  **Deserialization:** Re-created by the engine's codec system when loading a player's data from a persistent store. The static CODEC field dictates this process.
-   **Scope:** An instance of NPCMemory persists as long as it is held within a PlayerMemories component. Its lifecycle is therefore tied to the player's session and their saved data. Temporary instances created by GatherMemoriesSystem for checking purposes are extremely short-lived, existing only for the duration of a single system tick.
-   **Destruction:** Instances are eligible for garbage collection when a player's data is unloaded and no longer referenced, or immediately after a temporary check within the GatherMemoriesSystem.

## Internal State & Concurrency
-   **State:** Mutable. The class contains fields such as capturedTimestamp and foundLocationZoneNameKey which are populated after the object is constructed. The processConfig method further mutates the internal state by resolving the correct translation key for the memory's title, introducing conditional logic into the object's final state.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be created, modified, and accessed exclusively within the single-threaded context of the server's main game loop, specifically by an Entity Component System. Unsynchronized access from other threads will lead to unpredictable behavior and data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | String | O(1) | Returns the unique identifier for this memory, which is the NPC's role name. |
| getTitle() | String | O(1) | Returns the I18n translation key for the memory's display name. |
| getIconPath() | String | O(1) | Provides the asset path for the UI icon representing this memory. |
| getLocationMessage() | Message | O(1) | Constructs a translatable Message object representing the location of discovery. |
| processConfig() | void | O(1) | Internal post-processing logic to resolve the final memory title key. **Warning:** This must be called after manual instantiation to ensure correct state. |

## Integration Patterns

### Standard Usage
This class is rarely manipulated directly. It is typically retrieved from a player's component data for display in a user interface.

```java
// Retrieve a player's memories and iterate through them
PlayerMemories playerMemories = player.getComponent(PlayerMemories.class);
for (Memory memory : playerMemories.getRecordedMemories()) {
    if (memory instanceof NPCMemory npcMemory) {
        // Use npcMemory to display UI elements
        String titleKey = npcMemory.getTitle();
        Message location = npcMemory.getLocationMessage();
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation After Storage:** Do not modify an NPCMemory object after it has been recorded in a PlayerMemories component. All state should be finalized before it is committed.
-   **Skipping `processConfig`:** When manually creating an instance, failing to call processConfig can result in incorrect or missing UI text for the memory's title. The codec handles this automatically during deserialization.

## Data Pipeline
NPCMemory does not process data itself; it *is* the data. It represents the output of a detection and collection process.

> Flow:
> Player Proximity Event -> **NPCMemory (Instantiation & Population)** -> PlayerMemories Component -> Persistence Layer

---
---
description: Architectural reference for NPCMemory.GatherMemoriesSystem
---

# NPCMemory.GatherMemoriesSystem

**Package:** com.hypixel.hytale.builtin.adventure.memories.memories.npc
**Type:** Entity Component System

## Definition
```java
// Signature
public static class GatherMemoriesSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
GatherMemoriesSystem is the behavioral counterpart to the NPCMemory data class. As an EntityTickingSystem, it embodies the core logic of the NPC memory discovery feature within the server's Entity Component System (ECS) framework.

Its fundamental responsibility is to monitor the proximity between players and NPCs. The system operates on a specific subset of entities defined by its QUERY, which targets player entities equipped with a PlayerMemories component. This targeted approach ensures the system only executes logic where it is relevant, which is a core tenet of ECS design.

For performance, the system leverages a SpatialResource, a specialized spatial partitioning data structure. This allows for highly efficient "nearby" queries, avoiding a costly iteration over every NPC in the world on every tick. Upon detecting a player near a memory-eligible NPC, the system orchestrates the entire collection workflow: it verifies if the memory is new, records it, sends a notification to the player, and spawns a temporary visual effect. All entity modifications are funneled through a CommandBuffer, a standard ECS pattern for ensuring deterministic and thread-safe state changes.

## Lifecycle & Ownership
-   **Creation:** Instantiated once during server startup or plugin initialization. It is registered with the world's system scheduler to be executed every tick.
-   **Scope:** Singleton per world instance. The system persists for the entire lifetime of the server world.
-   **Destruction:** De-registered and garbage collected during server shutdown.

## Internal State & Concurrency
-   **State:** Effectively stateless. Its only instance field, `radius`, is immutable after construction. The system does not store any data related to its execution between ticks; all necessary state is read directly from ECS components in the world.
-   **Thread Safety:** The `tick` method is executed by the engine's scheduler and is guaranteed to operate within a single-threaded context for each world update. It is fundamentally unsafe to invoke the `tick` method from any other thread. The use of a CommandBuffer for all world mutations is the primary mechanism that ensures state changes are applied safely and in a deterministic order at the end of the tick. The system also uses a thread-local list for spatial query results, reinforcing its design for single-threaded execution per tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmdBuffer) | void | O(k log n) | The main entry point called by the engine each frame for each matching entity. Complexity depends on the spatial query (n entities) and results (k). |
| getQuery() | Query | O(1) | Returns the static Query object that defines which entities this system will operate on. |

## Integration Patterns

### Standard Usage
This system is not used directly by developers. It is automatically registered and executed by the engine. Feature customization is achieved by configuring the properties of NPC roles (e.g., `isMemory`) and the MemoriesGameplayConfig, not by interacting with this system's code.

### Anti-Patterns (Do NOT do this)
-   **Direct Invocation:** Never call the `tick` method manually. Doing so bypasses the engine's scheduler and command buffer system, which will corrupt entity state and cause severe concurrency violations.
-   **Direct Instantiation:** Do not create instances of this system using `new`. It must be managed by the engine's system registry to function correctly.

## Data Pipeline
This system implements the primary data processing pipeline for discovering NPC memories.

> Flow:
> **Input:** Player TransformComponent (Position)
> 1.  Perform spatial query using player position to find nearby NPC entities.
> 2.  Filter NPCs to find those with a memory-enabled Role.
> 3.  For each valid NPC, construct a temporary NPCMemory object.
> 4.  Query the player's PlayerMemories component to check for pre-existing memory.
> 5.  **If New:**
>     a. Finalize the NPCMemory object with timestamp and location data.
>     b. Mutate PlayerMemories component by adding the new memory.
>     c. Enqueue a command to send a network notification to the player.
>     d. Enqueue a command to spawn a temporary item entity for visual feedback.
> **Output:** Modified PlayerMemories component; new item entity; client-bound network packets.

