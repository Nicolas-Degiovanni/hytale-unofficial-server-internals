---
description: Architectural reference for RefillContainerInteraction
---

# RefillContainerInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Configuration Object

## Definition
```java
// Signature
public class RefillContainerInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The RefillContainerInteraction is a server-side, data-driven class that defines the logic for an entity filling a container item from a fluid source in the world. It is a specialized implementation within the broader interaction system, inheriting its base contract from SimpleBlockInteraction.

Architecturally, this class serves as a bridge between game configuration and the live game simulation. Its primary role is to execute a state change on an item stack based on the fluid type of a targeted block. For example, it governs the transformation of an "empty bucket" item into a "water bucket" item when a player interacts with a water block.

The entire behavior—which fluids are allowed, what the resulting item is, and whether the source fluid is consumed—is defined declaratively and loaded via the Hytale Codec system. This allows designers to create complex container-filling mechanics without requiring new Java code, simply by defining the interaction's properties in an asset file.

## Lifecycle & Ownership
- **Creation:** Instances of RefillContainerInteraction are not created dynamically during gameplay. They are instantiated and deserialized from configuration files by the server's Codec and Asset systems during the server bootstrap sequence.
- **Scope:** An instance persists for the entire server session. It is treated as an immutable configuration definition once loaded into memory.
- **Destruction:** The object is garbage collected along with all other server assets when the server process is shut down.

## Internal State & Concurrency
- **State:** The primary state, defined in the `refillStateMap`, is **immutable** after initial deserialization. However, the class employs a lazy initialization pattern for performance, caching derived data in the `allowedFluidIds` and `fluidToState` fields. These caches are mutable and populated on first access.

- **Thread Safety:** This class is **not thread-safe**. The lazy initialization of its internal caches is not protected by synchronization primitives. It is designed under the assumption that it will be fully decoded and initialized on the main server thread during startup. All subsequent calls to its methods, particularly `interactWithBlock`, are expected to occur synchronously within a single world-tick thread.

    **WARNING:** Accessing the internal state or invoking methods from multiple threads concurrently will lead to race conditions and unpredictable server behavior. Do not share instances of this class across different world threads without external locking.

## API Surface
The public contract is primarily consumed by the server's core InteractionModule.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(log N) | Executes the core refill logic. Complexity is dominated by world lookups and a binary search on N allowed fluids. Throws exceptions on invalid state. |
| generatePacket() | Interaction | O(1) | Constructs a network packet to inform the client about this interaction. |
| configurePacket(packet) | void | O(N) | Populates the network packet with the list of N allowed fluid IDs for client-side prediction or UI feedback. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly by module or gameplay developers. Instead, it is configured as part of an item's asset definition. The server's InteractionModule automatically discovers and executes this interaction when a player holding the configured item interacts with a block.

The system's internal invocation resembles the following conceptual flow:

```java
// Conceptual example of how the InteractionModule uses this class.
// This code does not exist in this form; it is for illustration only.

// 1. An interaction event is received for an entity.
InteractionContext context = createInteractionContextFor(entity, targetBlock);
ItemStack heldItem = context.getHeldItem();

// 2. The system retrieves the interaction configuration for the held item.
RefillContainerInteraction interaction = heldItem.getItem().getInteraction(RefillContainerInteraction.class);

// 3. If the interaction is valid, it is executed.
if (interaction != null) {
    interaction.interactWithBlock(world, commandBuffer, type, context, heldItem, targetBlock, cooldownHandler);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new RefillContainerInteraction()`. The object is fundamentally data-driven and will be non-functional without being properly populated by the Codec system from a configuration file.
- **State Mutation:** Do not attempt to modify the `refillStateMap` or other fields after the server has loaded. Doing so will desynchronize the internal caches and cause severe logic errors.
- **Client-Side Simulation:** The `simulateInteractWithBlock` method is intentionally empty. This interaction is fully server-authoritative. Do not attempt to build client-side prediction logic around it, as the server holds the final say on all inventory and world state changes.

## Data Pipeline
The flow of data for a successful refill interaction is orchestrated by the server and is entirely authoritative.

> Flow:
> Player Input (Client) → Network Packet (Interaction) → Server InteractionModule → **RefillContainerInteraction.interactWithBlock** → CommandBuffer (Inventory & World Transactions) → World State Change → Network Sync (Inventory & Block Updates) → Client Render

---

