---
description: Architectural reference for PlayerVelocityInstructionSystem
---

# PlayerVelocityInstructionSystem

**Package:** com.hypixel.hytale.server.core.universe.system
**Type:** System Component

## Definition
```java
// Signature
public class PlayerVelocityInstructionSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The PlayerVelocityInstructionSystem is a specialized system within the server's Entity Component System (ECS) framework. Its primary architectural role is to act as a **translator and network bridge** for player entity movement. It does not calculate or apply physics; instead, it consumes abstract movement commands and dispatches them to the appropriate game client.

This system operates on a specific subset of entities: those that possess both a **PlayerRef** component (identifying them as a player-controlled entity) and a **Velocity** component (managing their movement state).

Its position in the execution order is critical. By declaring dependencies to run *after* generic velocity modifiers but *before* the core physics integration systems, it ensures that it processes a complete, consolidated set of velocity changes for a given tick. This prevents partial or out-of-order updates from being sent to the client, which could cause stuttering or desynchronization. In essence, it is the final authority that communicates the server's intended velocity changes for a player to that player's client.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's central ECS System Runner during world initialization. The engine discovers and registers the system automatically. It is not created on-demand.
- **Scope:** The instance persists for the entire lifetime of the World it is associated with. A single instance processes all relevant player entities within that world.
- **Destruction:** The instance is marked for garbage collection when the parent World is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** This system is fundamentally **stateless** between ticks. Its behavior is driven exclusively by the component data of the entities it processes. The internal query and dependency set are immutable after construction.
- **Thread Safety:** This system is **not thread-safe**. It is designed to be executed by a single-threaded System Runner as part of the main server game loop. All component data access (e.g., from ArchetypeChunk) is assumed to be externally synchronized by the ECS framework. Concurrent manual invocation of the tick method will lead to race conditions and undefined behavior.

## API Surface
The public contract is defined by its implementation of the EntityTickingSystem interface. Direct calls to these methods are prohibited; they are for framework use only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, buffer) | void | O(N) | Framework-invoked method. Processes a single entity's velocity instructions. N is the number of instructions. |
| getDependencies() | Set | O(1) | Returns the scheduling dependencies, defining its place in the execution order. |
| getQuery() | Query | O(1) | Returns the component query used by the framework to identify target entities. |

## Integration Patterns

### Standard Usage
A developer should never interact with this system directly. The correct pattern is to modify the **Velocity** component of a player entity. The system will automatically detect and process these changes on the subsequent server tick.

```java
// CORRECT: Add a velocity instruction to a player entity's component.
// The PlayerVelocityInstructionSystem will process this automatically.

// Assume 'playerEntity' is a valid entity with a Velocity component.
Velocity velocityComponent = playerEntity.getComponent(Velocity.class);

if (velocityComponent != null) {
    Vector3d jumpImpulse = new Vector3d(0, 10, 0);
    Velocity.Instruction instruction = Velocity.Instruction.add(jumpImpulse);
    
    // This is the integration point.
    velocityComponent.addInstruction(instruction);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PlayerVelocityInstructionSystem()`. The ECS framework is solely responsible for its lifecycle. Direct instantiation will result in a non-functional object that is not registered with the game loop.
- **Manual Invocation:** Never call the `tick` method manually. Doing so bypasses the ECS scheduler, query filters, and thread-safety guarantees, which will corrupt entity state.
- **Clearing Instructions Externally:** The system is responsible for clearing the instruction list on the Velocity component after processing. Clearing it from another system can cause instructions to be lost, effectively creating a race condition.

## Data Pipeline
This system acts as a specific, late-stage step in the server's data flow for player movement, converting an internal game state change into a network-bound message.

> Flow:
> Other Game System -> Adds `Velocity.Instruction` to `Velocity` Component -> **PlayerVelocityInstructionSystem** (Reads Instructions) -> Creates `ChangeVelocity` Packet -> PlayerRef Packet Handler -> Network Layer -> Player Client

