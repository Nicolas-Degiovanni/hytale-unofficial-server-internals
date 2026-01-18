---
description: Architectural reference for ItemRepairPageSupplier
---

# ItemRepairPageSupplier

**Package:** com.hypixel.hytale.server.core.modules.interaction.suppliers
**Type:** Transient Factory

## Definition
```java
// Signature
public class ItemRepairPageSupplier implements OpenCustomUIInteraction.CustomPageSupplier {
```

## Architecture & Concepts
The ItemRepairPageSupplier is a specialized, data-driven factory component. Its sole purpose is to act as a bridge between a generic server interaction, specifically an OpenCustomUIInteraction, and the creation of a concrete ItemRepairPage user interface.

This class is not a long-lived service. Instead, it is designed to be instantiated from game data files (e.g., JSON asset definitions) via its static CODEC. This pattern allows game designers to define interaction points in the world, such as an anvil, and configure their behavior—in this case, the penalty for repairing an item—without modifying engine source code.

When a player triggers the associated interaction, the game engine invokes the tryCreate method on the configured supplier instance. The supplier then validates the context (e.g., ensuring the player is holding an item) and, if successful, constructs and returns a new ItemRepairPage. This page is then managed by the player's own state machine.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale serialization system (the CODEC) when loading interaction definitions from game assets. It is never instantiated directly in game logic code.
- **Scope:** Extremely short-lived and event-scoped. An instance is held by a parent OpenCustomUIInteraction configuration object. When that interaction is triggered, the supplier is used once and then becomes eligible for garbage collection. It does not persist beyond the scope of the single interaction event.
- **Destruction:** Managed by the Java Garbage Collector. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
- **State:** The internal state, primarily the repairPenalty, is mutable only during deserialization by the CODEC. After instantiation, the object should be treated as immutable. It serves as a read-only container for its configured parameters.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, used, and discarded within the confines of a single server thread that processes player interactions. Sharing instances across threads would be a severe design violation and lead to unpredictable behavior.

## API Surface
The public contract is defined by the CustomPageSupplier interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tryCreate(ref, accessor, playerRef, context) | CustomUIPage | O(1) | Factory method. Validates the InteractionContext and attempts to construct a new ItemRepairPage. Returns null if the context is invalid, such as when the player is not holding an item. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly through Java code. Instead, it is specified declaratively within a game asset file that defines an interaction. The engine handles the lifecycle and invocation.

Conceptual Example (e.g., in a JSON or HOCON file):
```json
{
  "interactionType": "OpenCustomUIInteraction",
  "pageSupplier": {
    "type": "ItemRepairPageSupplier",
    "RepairPenalty": 0.25
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ItemRepairPageSupplier()`. The class is useless without its state being populated by the CODEC from a data source.
- **State Mutation:** Do not attempt to get an instance and modify its `repairPenalty` field at runtime. This breaks the data-driven design and can lead to inconsistent behavior. Configuration should be treated as immutable once loaded.
- **Caching or Re-use:** Do not cache and re-use instances of this supplier. Its lifecycle is intentionally tied to the interaction event.

## Data Pipeline
The ItemRepairPageSupplier functions as a specific factory node within the server's player interaction data flow.

> Flow:
> Game Asset (JSON) -> CODEC Deserializer -> In-memory `OpenCustomUIInteraction` config -> Player triggers interaction -> **`ItemRepairPageSupplier.tryCreate()`** -> New `ItemRepairPage` instance -> Player's active UI page state is updated

