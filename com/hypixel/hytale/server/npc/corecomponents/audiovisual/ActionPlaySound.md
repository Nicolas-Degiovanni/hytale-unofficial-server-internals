---
description: Architectural reference for ActionPlaySound
---

# ActionPlaySound

**Package:** com.hypixel.hytale.server.npc.corecomponents.audiovisual
**Type:** Transient Command

## Definition
```java
// Signature
public class ActionPlaySound extends ActionBase {
```

## Architecture & Concepts
ActionPlaySound is a concrete implementation of the Command Pattern within the Hytale NPC Behavior System. It represents a single, atomic, and fire-and-forget operation: instructing an NPC entity to play a sound effect.

This class acts as a bridge between the high-level NPC Behavior Tree engine and the low-level server audio system. It is not responsible for loading audio assets or managing the audio engine itself. Its sole responsibility is to dispatch a pre-resolved sound event, identified by an integer index, to be played at the NPC's current world position.

Instances of ActionPlaySound are designed to be configured declaratively in external NPC behavior assets (e.g., JSON files). A corresponding builder, BuilderActionPlaySound, parses the asset and instantiates this class, resolving the human-readable sound name into an efficient integer index during the server's asset loading phase.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by its corresponding builder, BuilderActionPlaySound, during the server's asset loading phase. The builder resolves a sound name from an NPC asset file into a final integer index, which is injected into the ActionPlaySound constructor.
- **Scope:** The object is effectively a stateless singleton for a given NPC behavior definition. It persists as long as the NPC behavior asset is loaded in memory. The same instance is reused for all NPCs that execute this specific action from their shared behavior tree.
- **Destruction:** The object is eligible for garbage collection when the server unloads the associated NPC behavior assets, typically during a server shutdown or a hot-reload of game data.

## Internal State & Concurrency
- **State:** Immutable. The only internal state is the final integer field soundEventIndex, which is set at construction time. This immutability is critical for allowing a single instance to be safely reused across multiple NPC entities and behavior tree executions.
- **Thread Safety:** This class is inherently thread-safe due to its immutable state. However, the execution context is strictly controlled. The execute method is designed to be called **only** from the main server thread as part of the game loop's entity update tick. The Store parameter passed to execute is not thread-safe and must not be accessed from other threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Triggers the sound event. Retrieves the NPC's position from its TransformComponent and dispatches a request to the SoundUtil. Always returns true to signal immediate completion to the behavior tree. |

**WARNING:** The execute method uses assertions to enforce that the target entity possesses both a TransformComponent and an NPCEntity component. Execution will fail with an AssertionError if these components are missing, indicating a critical entity configuration error.

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is a component of the NPC behavior system, configured in asset files and invoked automatically by the Behavior Tree engine. The following example illustrates how the engine would invoke the action.

```java
// Conceptual example of engine-level invocation
// An NPC's behavior tree node would call this method during its update tick.

// Assume 'action' is a pre-loaded ActionPlaySound instance
// Assume 'entityRef' is a reference to the active NPC
boolean result = action.execute(entityRef, currentRole, sensorData, deltaTime, worldStore);

// The engine uses the 'true' result to advance the behavior tree state.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ActionPlaySound()`. The soundEventIndex must be resolved from game assets via the BuilderActionPlaySound during server startup. Manual instantiation will result in an incorrect or missing sound index.
- **Stateful Subclassing:** Do not extend this class to add mutable state. Actions in the behavior system are designed to be stateless, reusable commands. Introducing state will break instance sharing and lead to unpredictable behavior across multiple NPCs.
- **Blocking Operations:** The execute method must complete within a single server tick. Do not introduce any blocking I/O, long-running computations, or asynchronous logic that would stall the main game loop.

## Data Pipeline
The flow of data from configuration to execution is a one-way pipeline from asset definition to client-side audio playback.

> Flow:
> NPC Behavior Asset (JSON) -> Asset Loading System -> BuilderActionPlaySound -> **ActionPlaySound** -> Behavior Tree Engine -> SoundUtil -> Server Network Subsystem -> Game Client Audio Engine

