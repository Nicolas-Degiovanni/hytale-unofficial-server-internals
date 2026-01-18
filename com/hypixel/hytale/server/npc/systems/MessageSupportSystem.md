---
description: Architectural reference for MessageSupportSystem
---

# MessageSupportSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Singleton (per World instance)

## Definition
```java
// Signature
public abstract class MessageSupportSystem<T extends MessageSupport> extends SteppableTickingSystem {
```

## Architecture & Concepts
The MessageSupportSystem is an abstract base class within the server-side Entity Component System (ECS) framework, designed to manage the lifecycle of timed events and signals on entities. It acts as a generic state processor for components that extend the MessageSupport contract.

Its primary function is to "tick down" the lifetime of active messages attached to an entity. This provides a decoupled, performant mechanism for implementing temporary states, buffs, or event flags without requiring each game logic system to manage its own timers.

This class embodies a **Strategy Pattern** through its concrete subclasses (e.g., BeaconSystem, NPCBlockEventSystem). Each subclass provides a specific *strategy* for a particular type of message component (BeaconSupport, NPCBlockEventSupport), while the abstract parent class provides the core, reusable lifecycle logic. This design allows the engine to process many different kinds of timed events using a single, optimized architectural pattern.

The system operates exclusively on the server and is a fundamental part of the NPC and entity interaction model.

## Lifecycle & Ownership
- **Creation:** Instances of MessageSupportSystem subclasses are not created manually. They are discovered via reflection and instantiated by the server's ECS SystemGraph during world initialization. The framework is responsible for providing the necessary ComponentType and dependency information to the constructor.
- **Scope:** A single instance of each concrete subclass (e.g., BeaconSystem) exists for the entire lifetime of a server world. It is a long-lived, session-scoped object.
- **Destruction:** The system is destroyed and eligible for garbage collection only when the corresponding server world is shut down and the entire ECS graph is torn down.

## Internal State & Concurrency
- **State:** The MessageSupportSystem is **stateless**. It holds immutable references to its configuration (the ComponentType and its dependencies) but does not maintain any mutable state between ticks. All state it operates on is contained within the components of the entities it processes.
- **Thread Safety:** This system is designed for parallel execution and is considered **thread-safe** within the context of the ECS scheduler. The `isParallel` method delegates the decision, allowing the scheduler to run `steppedTick` for different entity chunks concurrently on multiple threads. Because the system is stateless and only modifies the component data for the single entity it is given in each `steppedTick` call, data races are prevented by the scheduler's architecture.

## API Surface
The public API is intended for consumption by the ECS framework, not by game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| steppedTick(dt, index, chunk, store, buffer) | void | O(N) | **Engine Callback.** Processes a single entity. Iterates N active message slots, ticking their age and deactivating them upon expiry. |
| getQuery() | Query | O(1) | **Engine Callback.** Returns the component query that determines which entities this system will operate on. |
| getDependencies() | Set | O(1) | **Engine Callback.** Declares dependencies on other systems to the scheduler to ensure correct execution order. |

## Integration Patterns

### Standard Usage
Direct interaction with this system is not possible. A developer uses this system implicitly by adding a relevant component to an entity's archetype. The system will automatically discover and process the entity.

```java
// Conceptual Example: Adding a component to an entity
// This is typically done via data-driven archetype definitions or a CommandBuffer.

// Get the component store for NPCBlockEventSupport
Store<EntityStore> supportStore = world.getStore(NPCBlockEventSupport.class);
NPCBlockEventSupport component = supportStore.get(myEntity);

// Create and activate a message that will expire in 5.0 seconds
NPCMessage message = new NPCMessage(5.0f);
component.getMessageSlots()[0] = message;
message.activate();

// The NPCBlockEventSystem will now automatically manage this message's lifecycle.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BeaconSystem()`. The ECS framework must control the system's lifecycle and dependency injection. Manual creation will result in a non-functional system that is not registered with the engine.
- **Manual Invocation:** Never call `steppedTick` directly. Doing so bypasses the ECS scheduler, breaking execution order guarantees and thread safety. This can lead to severe race conditions, state corruption, and unpredictable crashes.

## Data Pipeline
This system acts as a state processor within the main server tick loop. It does not ingest external data but rather transforms the state of components over time.

> Flow:
> Other Game System (e.g., AI, Interaction) -> Activates NPCMessage in a Component -> **MessageSupportSystem** (on next tick) -> Updates message age -> Deactivates NPCMessage on expiry -> Other Game System -> Reads deactivated state

---
# MessageSupportSystem.BeaconSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Singleton (per World instance)

## Definition
```java
// Signature
public static class BeaconSystem extends MessageSupportSystem<BeaconSupport> {
```

## Architecture & Concepts
This is a concrete implementation of the MessageSupportSystem. It is specialized to operate exclusively on entities that possess the **BeaconSupport** component.

Its role within the engine is to manage the lifecycle of beacon-related signals. For example, it could be used to time out the effect a beacon has on nearby NPCs or to manage the duration of a temporary claim. By specializing the generic system, the engine maintains a clean separation of concerns while reusing the core timing logic.

All architectural concepts, lifecycle, and concurrency guarantees from the parent MessageSupportSystem apply directly to this class.

## Integration Patterns

### Standard Usage
To use this system, an entity must be given a BeaconSupport component. Game logic can then activate messages within that component's slots to create timed beacon effects.

```java
// Conceptual: Making an entity respond to a beacon for 30 seconds.
Entity myNpc = ...;
CommandBuffer commands = ...;

// Add the component if it doesn't exist
BeaconSupport support = new BeaconSupport();
commands.addComponent(myNpc, support);

// Activate a message that the BeaconSystem will manage
support.getMessageSlots()[0] = new NPCMessage(30.0f);
support.getMessageSlots()[0].activate();
```

