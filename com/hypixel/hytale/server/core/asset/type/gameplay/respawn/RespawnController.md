---
description: Architectural reference for RespawnController
---

# RespawnController

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay.respawn
**Type:** Strategy Interface

## Definition
```java
// Signature
public interface RespawnController {
```

## Architecture & Concepts
The RespawnController interface defines a strategic contract for handling player respawn logic on the server. It abstracts the specific rules and mechanics of how a player is brought back into the world after death, decoupling the core death processing system from the varied gameplay modes Hytale supports.

This component is a key part of Hytale's data-driven gameplay architecture. The static CODEC field, a CodecMapCodec, indicates that concrete implementations of this interface are registered and can be selected and configured via game assets. This allows designers to create unique respawn behaviors (e.g., spawn at world origin, spawn at a set home point, spawn with penalties) without modifying core engine code.

The controller operates directly on fundamental server-side objects like the World and EntityStore, granting it significant authority to mutate the game state. Its primary function is to determine a valid spawn location and issue the necessary commands to reposition or recreate the player entity.

## Lifecycle & Ownership
- **Creation:** Concrete implementations of RespawnController are not instantiated directly. They are loaded and deserialized from game asset files by the server's asset management system, using the registered CODEC. An instance is typically part of a larger gameplay ruleset asset.
- **Scope:** The lifecycle of a RespawnController instance is tied to the gameplay ruleset of the World it governs. It is generally stateless and can be treated as a singleton for a given ruleset, persisting as long as those rules are active.
- **Destruction:** The object is eligible for garbage collection when the World or its associated gameplay configuration is unloaded from memory, for example, during a server shutdown or a dynamic world change.

## Internal State & Concurrency
- **State:** As an interface, RespawnController defines no state. Implementations are **strongly expected** to be stateless. All necessary context for a respawn operation (the world, the player entity, and a command buffer) is passed as arguments to the respawnPlayer method. This ensures that a single controller instance can be safely reused for all players governed by the same ruleset.
- **Thread Safety:** Implementations of this interface are **not thread-safe** and must not be treated as such. The respawnPlayer method performs critical state mutations on the World and EntityStore. It must be invoked exclusively from the main server thread that owns the target World instance to prevent catastrophic race conditions and world state corruption. The mandatory use of a CommandBuffer enforces that mutations are queued and executed deterministically within the server's main tick loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | CodecMapCodec | O(1) | Static codec for serializing and deserializing different RespawnController implementations from data. |
| respawnPlayer(World, Ref, CommandBuffer) | void | O(N) | Executes the respawn logic. Complexity depends on the implementation's world-querying strategy for finding a spawn point. |

## Integration Patterns

### Standard Usage
The RespawnController is invoked by a higher-level system, such as a PlayerLifecycleManager or DeathSystem, in response to a player death event. The system retrieves the appropriate controller for the current game mode and passes the required world context.

```java
// Pseudo-code demonstrating invocation within a server system
void onPlayerDeath(Player player) {
    World world = player.getWorld();
    RespawnController controller = world.getGameplayRules().getRespawnController();
    
    // The CommandBuffer ensures all mutations are deferred to the main tick
    CommandBuffer<EntityStore> commands = world.getCommandBuffer();
    Ref<EntityStore> playerEntityRef = player.getEntityStoreRef();

    controller.respawnPlayer(world, playerEntityRef, commands);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of a RespawnController implementation with the new keyword. The engine relies on the CODEC system for loading and networking these strategies. Bypassing it will lead to deserialization failures and inconsistent game state.
- **Asynchronous Execution:** Do not invoke respawnPlayer from a separate thread or an asynchronous task. All interactions with the World and its components must be synchronized with the main server tick.
- **Direct State Mutation:** Do not modify the World or EntityStore directly within an implementation. All state changes **must** be queued onto the provided CommandBuffer to maintain deterministic execution and prevent mid-tick corruption.

## Data Pipeline
The controller acts as a processing node in the player death-to-respawn data flow.

> Flow:
> Player Death Event -> Server Gameplay System -> **RespawnController.respawnPlayer()** -> CommandBuffer -> World Tick Execution -> Player Entity State Updated

