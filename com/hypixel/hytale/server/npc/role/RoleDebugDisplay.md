---
description: Architectural reference for RoleDebugDisplay
---

# RoleDebugDisplay

**Package:** com.hypixel.hytale.server.npc.role
**Type:** Transient

## Definition
```java
// Signature
public class RoleDebugDisplay {
```

## Architecture & Concepts
The RoleDebugDisplay class is a server-side diagnostic utility responsible for aggregating and rendering real-time debug information for Non-Player Character (NPC) entities. It functions as a specialized data presenter within the server's Entity Component System (ECS) framework.

Its primary architectural role is to act as a bridge between the raw state data stored in various disparate components (e.g., AI state, physics, entity stats, world light levels) and a human-readable visual output. This output is injected back into the ECS as a Nameplate component, effectively displaying the debug information above the NPC in the game world.

This class is not part of the core AI logic; it is a passive observer that reads entity state via ArchetypeChunks and requests state modifications (adding or updating a Nameplate) through a CommandBuffer. This one-way data flow ensures that debugging operations do not introduce side effects into the systems they are observing.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively through the static factory method `create`. This method takes an EnumSet of RoleDebugFlags which dictates which pieces of information the instance will be configured to display. If the provided set of flags is empty, the factory method returns null, preventing the creation of a useless object.

- **Scope:** A RoleDebugDisplay object is a short-lived, transient object. It is typically created at the beginning of a system's update tick, used to process a batch of entities within that single tick, and then becomes eligible for garbage collection. The internal state (specifically the StringBuilder) is reset after each entity is processed, allowing a single instance to be reused for all relevant NPCs within one update cycle.

- **Destruction:** The object holds no native resources and does not require explicit cleanup. It is managed entirely by the Java garbage collector once all references to it are dropped, typically after the system's update method completes.

## Internal State & Concurrency
- **State:** The class maintains two forms of internal state:
    1.  **Configuration State:** A series of boolean flags (e.g., debugDisplayState, debugDisplayHP) are set once upon construction by the `create` factory method. This state is immutable for the lifetime of the object.
    2.  **Working State:** A single, mutable StringBuilder instance is used to construct the debug string for each entity. This buffer is cleared at the end of every call to the `display` method, making the object stateful but reusable within a single-threaded loop.

- **Thread Safety:** **CRITICAL:** This class is not thread-safe and is designed for single-threaded access. The internal StringBuilder is mutated without any synchronization. Concurrent calls to the `display` method on the same instance will result in corrupted output and race conditions. It must only be used from the context of the server's main game loop or a specific ECS system's update thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| create(EnumSet) | RoleDebugDisplay | O(1) | Static factory. Constructs and configures a new instance based on the provided flags. Returns null if no flags are active. |
| display(Role, int, ArchetypeChunk, CommandBuffer) | void | O(1) | Gathers data for a single entity, builds the debug string, and enqueues a command to update the entity's Nameplate. |

## Integration Patterns

### Standard Usage
The intended pattern is for a server-side system to create an instance of RoleDebugDisplay if debugging is enabled. The system then iterates over its entities and calls the `display` method for each one, passing the necessary ECS context.

```java
// Within an ECS System's update method
EnumSet<RoleDebugFlags> flags = getServerDebugConfig().getNpcDebugFlags();
RoleDebugDisplay debugDisplay = RoleDebugDisplay.create(flags);

// Do not proceed if no debug flags are enabled
if (debugDisplay == null) {
    return;
}

// Iterate over all entities managed by this system
for (ArchetypeChunk<EntityStore> chunk : query.getArchetypeChunks()) {
    for (int i = 0; i < chunk.getCount(); i++) {
        Role role = chunk.getComponent(i, Role.getComponentType());
        // Pass context for the current entity to the display method
        debugDisplay.display(role, i, chunk, commandBuffer);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to create an instance using reflection. Always use the static `create` factory method.
- **Ignoring Null Return:** The `create` method can return null. Failure to check for this will result in a NullPointerException if no debug flags are enabled.
- **Concurrent Access:** Do not share a RoleDebugDisplay instance across multiple threads. It must be confined to the thread that created it.
- **State Leakage:** Do not manually manipulate the internal StringBuilder. The `display` method manages its lifecycle completely.

## Data Pipeline
The class operates as a data processing node within the server's main loop, transforming raw component data into a formatted string.

> Flow:
> Server Configuration (RoleDebugFlags) -> **RoleDebugDisplay.create()** -> Instance is passed to an ECS System -> System iterates entities and calls **display()** -> Method reads multiple Components (Transform, Velocity, EntityStatMap, etc.) from ArchetypeChunk -> String is built -> CommandBuffer is used to add/update a Nameplate Component on the entity.

