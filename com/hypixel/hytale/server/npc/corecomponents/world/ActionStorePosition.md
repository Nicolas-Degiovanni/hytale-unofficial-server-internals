---
description: Architectural reference for ActionStorePosition
---

# ActionStorePosition

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient Component

## Definition
```java
// Signature
public class ActionStorePosition extends ActionBase {
```

## Architecture & Concepts
ActionStorePosition is a concrete implementation of the Command Pattern, forming a fundamental building block within the server-side NPC artificial intelligence system. Its primary architectural role is to act as a data retrieval mechanism, bridging an NPC's persistent memory with its immediate, per-tick sensory context.

This action retrieves a pre-saved world coordinate from an NPC-specific storage slot and injects it into the current operational context, represented by the InfoProvider. This allows an NPC to "remember" a significant location—such as a home base, a patrol waypoint, or the last known position of a target—and make it available for subsequent actions like pathfinding or aiming.

It is designed to be a small, stateless, and highly reusable component within a larger AI structure, such as a Behavior Tree or a Finite State Machine.

### Lifecycle & Ownership
- **Creation:** This object is instantiated exclusively by the server's asset loading pipeline, specifically through its corresponding builder, BuilderActionStorePosition. The configuration, including the critical **slot** index, is defined within NPC asset files (e.g., JSON behavior definitions). It is never created manually during gameplay.
- **Scope:** The lifetime of an ActionStorePosition instance is tied to the loaded NPC behavior definition. It is a stateless command object that is re-executed whenever the NPC's AI logic dictates, but the object itself persists as long as the NPC's behavior asset is loaded in memory.
- **Destruction:** The object is marked for garbage collection when the server unloads the associated NPC asset bundle, typically during a server shutdown or a dynamic asset reload.

## Internal State & Concurrency
- **State:** ActionStorePosition is effectively immutable. Its only internal state is the final integer field **slot**, which is initialized once at construction and cannot be changed. The class does not cache any world state or session data.
- **Thread Safety:** The class itself is inherently thread-safe due to its immutable design. However, it operates on mutable, thread-unsafe game state objects passed as method parameters (Ref, Role, InfoProvider, Store). The server's core architecture must enforce a single-threaded execution model for each individual NPC's AI tick to prevent race conditions on the game state it manipulates. This component contains no internal locking mechanisms.

## API Surface
The public contract is defined by its parent, ActionBase, and is focused on execution within the server's AI tick.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Precondition check. Returns true only if the sensory context (InfoProvider) exists and contains a position provider. |
| execute(...) | boolean | O(1) | Performs the core logic. Retrieves a Vector3 from the Role's storage and writes it into the InfoProvider. Always returns true. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is configured declaratively within an NPC's behavior asset and executed automatically by the server's AI engine. The engine is responsible for passing the correct context during an AI update tick.

A conceptual representation of its invocation by the system:

```java
// Executed by an internal AI processing loop (e.g., Behavior Tree Node)
// The 'action' instance is retrieved from the NPC's loaded behavior asset.
ActionStorePosition action = npc.getBehavior().getAction("rememberHomePosition");

// The engine first validates if the action can run in the current context
if (action.canExecute(entityRef, role, sensorInfo, deltaTime, store)) {
    // If validation passes, the engine executes the action
    action.execute(entityRef, role, sensorInfo, deltaTime, store);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using **new ActionStorePosition()**. The object will lack the correct **slot** configuration and will not be part of the managed AI lifecycle. All instances must originate from the asset pipeline.
- **Ignoring Preconditions:** Calling **execute** without a preceding successful call to **canExecute** can lead to runtime exceptions or unpredictable behavior if the required **InfoProvider** context is missing. The AI engine's contract is to always perform this check.

## Data Pipeline
ActionStorePosition acts as a data retrieval and injection point. It does not transform data but rather moves it from long-term storage to a short-term operational context.

> Flow:
> NPC AI Engine Tick -> **ActionStorePosition.execute()** -> Accesses Role's MarkedEntitySupport -> Retrieves Stored Position Vector -> Injects Position into InfoProvider -> Subsequent AI Actions (e.g., ActionPathToPosition) consume the new position data.

