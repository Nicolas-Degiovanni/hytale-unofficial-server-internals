---
description: Architectural reference for INonPlayerCharacter
---

# INonPlayerCharacter

**Package:** com.hypixel.hytale.server.core.universe.world.npc
**Type:** Interface

## Definition
```java
// Signature
public interface INonPlayerCharacter {
```

## Architecture & Concepts
The INonPlayerCharacter interface is the fundamental contract for all non-player character entities within the server's world simulation. It serves as a marker and a minimal API for the engine to distinguish NPCs from other entity types, such as players or items.

This abstraction is critical for decoupling core engine systems from concrete NPC implementations. Systems like the AI scheduler, NPC Spawner, and the persistence layer operate against this interface, allowing them to manage any type of NPC without knowledge of its specific behaviors (e.g., a passive Villager versus a hostile Golem). The interface guarantees that every NPC can provide a stable, serializable identifier for its type, which is essential for saving, loading, and network replication.

## Lifecycle & Ownership
As an interface, INonPlayerCharacter has no lifecycle itself. The following applies to all objects that **implement** this interface.

-   **Creation:** Concrete NPC instances are almost exclusively instantiated by the NPCFactory service. This factory is responsible for constructing the correct NPC class based on a given type ID, often during world generation, spawner activation, or in response to game events. Direct instantiation of NPC classes is strongly discouraged.
-   **Scope:** An NPC's lifecycle is tightly bound to its existence within a World instance. It persists as long as the NPC is alive and its containing chunk remains loaded in memory.
-   **Destruction:** An instance is marked for garbage collection when the NPC is killed, despawned due to game rules (e.g., distance from players), or when its parent chunk is unloaded from the server.

## Internal State & Concurrency
-   **State:** This interface is stateless. However, any class implementing it is guaranteed to be highly stateful, managing complex data such as health, position, velocity, AI state machine, and inventory.
-   **Thread Safety:** Implementations are **not thread-safe** and are designed for single-threaded access within their owning world's update tick. All modifications to an NPC's state must be queued and executed on the main server thread to prevent race conditions, state corruption, and physics simulation errors.

## API Surface
The public contract is minimal, focusing solely on type identification.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getNPCTypeId() | String | O(1) | Returns the unique, human-readable string identifier for the NPC type (e.g., "hytale:goblin_warrior"). |
| getNPCTypeIndex() | int | O(1) | Returns the integer-based index for the NPC type. This is a fast, optimized lookup for performance-critical systems. |

## Integration Patterns

### Standard Usage
Systems should use this interface to generically handle NPCs without coupling to specific implementations. The primary pattern is to check if a generic Entity is an NPC, and then use the interface to retrieve its type information.

```java
// A system processing entities in a chunk
for (Entity entity : chunk.getEntities()) {
    if (entity instanceof INonPlayerCharacter) {
        INonPlayerCharacter npc = (INonPlayerCharacter) entity;
        
        // Use the fast integer index for lookups in data tables
        NPCTemplate template = npcTemplateRegistry.get(npc.getNPCTypeIndex());
        
        // Use the string ID for logging or serialization
        log.info("Processing NPC of type: " + npc.getNPCTypeId());
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Casting to Concrete Types:** Avoid casting to a specific implementation like Goblin or Villager unless you are in a system explicitly designed to handle that type. Relying on concrete types breaks the abstraction and makes the system brittle.
-   **String Comparison in Hot Paths:** Do not use getNPCTypeId() for frequent checks or lookups inside the main game loop. String comparisons are significantly slower than integer comparisons. Always prefer getNPCTypeIndex() for performance-sensitive code.

