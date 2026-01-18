---
description: Architectural reference for SpawnNPCInteraction
---

# SpawnNPCInteraction

**Package:** com.hypixel.hytale.server.npc.interactions
**Type:** Configuration Object

## Definition
```java
// Signature
public class SpawnNPCInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The SpawnNPCInteraction class is a data-driven, server-side component that defines a specific game action: spawning an NPC when a player interacts with a block. It is a concrete implementation of the abstract **SimpleBlockInteraction**, designed to be configured within game asset files rather than being hardcoded.

Architecturally, it serves as a declarative bridge between the generic Block Interaction System and the specialized NPC Management System. Its existence is defined entirely by its static **CODEC** field, which uses the engine's serialization framework to deserialize an instance from a configuration file, typically associated with a specific **BlockType**.

The core responsibility of this class is to translate a player interaction event into a deferred entity creation command. It performs complex calculations to determine the final spawn position and rotation of the NPC, accounting for the parent block's orientation and applying configured offsets. This ensures that NPCs are spawned in a predictable and contextually appropriate manner relative to the block they originate from.

## Lifecycle & Ownership
- **Creation:** Instances of SpawnNPCInteraction are not created directly using the *new* keyword. They are instantiated by the server's asset loading system during bootstrap or hot-reloading. The static **CODEC** field is invoked to deserialize the object's properties from a corresponding asset definition.
- **Scope:** The lifetime of a SpawnNPCInteraction instance is tied to the asset that defines it, typically a **BlockType**. It persists in memory as part of the block's configuration for as long as that block type is loaded. It is effectively a stateless template for an action.
- **Destruction:** The object is eligible for garbage collection when its parent asset configuration is unloaded by the server.

## Internal State & Concurrency
- **State:** The internal state of this class (entityId, spawnOffset, spawnChance) is populated once during deserialization and is treated as immutable for the remainder of its lifecycle. It does not hold or manage any dynamic runtime state.

- **Thread Safety:** This class is designed to be thread-safe within the server's execution model.
    - The interaction methods are invoked by the world's main tick loop.
    - The use of **ThreadLocalRandom** prevents contention when calculating spawn chance.
    - **CRITICAL:** The actual NPC spawning logic is not executed immediately. It is wrapped in a lambda and submitted to a **CommandBuffer**. This is a crucial concurrency pattern that queues the entity creation operation to be executed safely on the main entity store thread, preventing race conditions and ensuring world state consistency.

## API Surface
The public contract is inherited from **SimpleBlockInteraction** and is invoked by the server's interaction handling system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(1) | Server-side entry point. Checks spawn chance and queues an NPC spawn command via the CommandBuffer. |
| simulateInteractWithBlock(...) | void | O(1) | Client-side prediction entry point. Queues a simulated NPC spawn command for immediate feedback. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly in plugin or gameplay code. The engine's interaction system identifies the component based on block configuration and calls it automatically. The following is a conceptual representation of how the engine might dispatch an event to this handler.

```java
// Engine-level code (conceptual)
// A player interacts with a block at 'targetBlock'

BlockType blockType = world.getBlockTypeAt(targetBlock);
SimpleBlockInteraction interaction = blockType.getInteractionHandler();

// The engine dispatches the event to the configured handler,
// which may be an instance of SpawnNPCInteraction.
if (interaction != null) {
    interaction.interactWithBlock(world, commandBuffer, type, context, item, targetBlock, cooldowns);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new SpawnNPCInteraction()`. The object will be unconfigured and useless, as its fields like entityId will be null. It must be created via the asset system's deserialization process.
- **Runtime State Modification:** Do not attempt to get an instance of this class and modify its fields at runtime. It is designed as a static configuration template.
- **Direct Invocation of spawnNPC:** The private **spawnNPC** method should not be made public or called directly. Bypassing the **interactWithBlock** method circumvents the **CommandBuffer** pattern, which can lead to severe concurrency issues and world corruption.

## Data Pipeline
The flow of data and control begins with player input and results in a change to the world's entity state.

> Flow:
> Player Input -> Network Packet -> Server Interaction System -> BlockType Configuration Lookup -> **SpawnNPCInteraction.interactWithBlock** -> CommandBuffer -> NPCPlugin.spawnNPC -> EntityStore Update -> Network State Sync to Clients

