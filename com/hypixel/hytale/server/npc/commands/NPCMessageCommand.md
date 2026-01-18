---
description: Architectural reference for NPCMessageCommand
---

# NPCMessageCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Transient

## Definition
```java
// Signature
public class NPCMessageCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The NPCMessageCommand class implements a server-side administrative command for interacting with Non-Player Characters (NPCs). It serves as a direct interface between a privileged user, such as a server administrator, and the NPC messaging subsystem.

This command is a node within the server's broader Command System. It does not contain business logic itself; rather, it acts as a translator, converting a text-based command from a player into a specific action within the Entity Component System (ECS). Its primary responsibility is to parse user-provided arguments, identify target entities, and invoke methods on the appropriate components.

The command operates in two distinct modes:
1.  **Single-Target Mode:** Dispatches a message to a single NPCEntity. The target is resolved using the player's current focus or a specified entity ID.
2.  **Broadcast Mode:** Dispatches a message to all entities in the world that possess a BeaconSupport component. This is achieved via a highly-optimized, parallelized query over the ECS data store.

Interaction with the game world is mediated entirely through the ECS interfaces, specifically the Store and Ref objects passed into the execute method. This design decouples the command logic from the underlying data layout of entities, ensuring that changes to NPC components do not break the command's functionality.

## Lifecycle & Ownership
- **Creation:** A single instance of NPCMessageCommand is instantiated by the server's core CommandSystem during the server bootstrap phase. The system discovers all classes extending AbstractPlayerCommand and registers them.
- **Scope:** The command object is a long-lived singleton that persists for the entire server session. It is designed to be stateless, acting as a template for execution.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server is shutting down and the CommandSystem is dismantled.

## Internal State & Concurrency
- **State:** The NPCMessageCommand object is effectively **immutable** after construction. Its member fields, such as messageArg and allArg, are argument descriptors that are configured once in the constructor and never modified. All state related to a specific invocation is passed into the execute method via the CommandContext parameter, ensuring that each execution is isolated.

- **Thread Safety:** The class instance itself is thread-safe due to its stateless nature. The execute method, however, performs operations on the shared world state. The use of `store.forEachEntityParallel` indicates that the broadcast mode performs a parallel read operation across multiple worker threads.

    **WARNING:** While the ECS is designed for concurrency, any modifications to components must be funneled through a CommandBuffer to prevent race conditions. This class correctly delegates state changes to the BeaconSupport component, which is responsible for its own thread safety. Direct mutation of components within the parallel loop is a critical anti-pattern.

## API Surface
The public contract is defined by its superclass, AbstractPlayerCommand. The primary entry point is the execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) or O(N) | Executes the command logic. Complexity is O(1) for a single target and O(N) when the *all* flag is used, where N is the number of entities with a BeaconSupport component. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by developers. It is invoked automatically by the server's command processing pipeline in response to a player's chat input.

The intended use is via the in-game console:
```
/npc message "Follow me!" --entity 12345 --expiration 10.0
/npc message "Patrol the area." --all
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new NPCMessageCommand()`. The CommandSystem manages the lifecycle of all command objects. Manually creating an instance will result in an object that is not registered with the server and cannot be executed.
- **Bypassing CommandContext:** Do not call the execute method directly. The CommandContext is constructed by the command parser and contains critical, validated argument data. Calling execute with a manually created context can lead to NullPointerExceptions and unstable server state.
- **Stateful Implementation:** Do not add mutable member fields to this class. Command objects must remain stateless to ensure thread safety and predictable behavior across multiple, concurrent executions.

## Data Pipeline
The flow of data for this command begins with user input and terminates with a state change in the target NPC's component data.

> Flow:
> Player Chat Input -> Server Network Handler -> Command Parser -> **NPCMessageCommand.execute()** -> ECS Query (`store.getComponent` or `store.forEachEntityParallel`) -> BeaconSupport.postMessage() -> NPC Behavior System

