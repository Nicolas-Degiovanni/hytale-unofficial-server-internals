---
description: Architectural reference for NPCStepCommand
---

# NPCStepCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Command Object

## Definition
```java
// Signature
public class NPCStepCommand extends AbstractWorldCommand {
```

## Architecture & Concepts

The NPCStepCommand is a server-side administrative command designed for debugging and controlling the simulation of Non-Player Characters (NPCs). It integrates into the server's command processing system by extending AbstractWorldCommand, granting it access to the world state and entity store upon execution.

Its primary architectural role is not to perform NPC logic directly, but to act as a **command issuer** within the server's Entity Component System (ECS). When executed, it does not mutate an NPC's state. Instead, it attaches a transient **StepComponent** to the target entity or entities.

This design decouples the command's invocation from the actual simulation logic. A separate, dedicated system (presumably an NPC processing or AI system) is responsible for querying for entities with a StepComponent on each tick, executing a single simulation step using the provided delta-time, and then removing the component.

Furthermore, the command strategically adds a **Frozen** component to the target NPC. This is a critical state management pattern that temporarily pauses the NPC's standard AI routines, preventing them from interfering with the manually triggered step. This ensures that the simulation step occurs in a controlled, predictable state.

## Lifecycle & Ownership

-   **Creation:** A single instance of NPCStepCommand is instantiated by the server's command registration service during the server bootstrap sequence. It is registered with the command handler under the name "step".
-   **Scope:** The object itself is stateless and persists for the entire server session. Its lifecycle is tied to the server's main lifecycle.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency

-   **State:** The NPCStepCommand class is **stateless**. Its member fields are final argument definitions (FlagArg, EntityWrappedArg) that are immutable after construction. All state required for execution, such as the command sender and world data, is provided via the CommandContext and Store parameters of the execute method.

-   **Thread Safety:** The command object is inherently thread-safe. The execution logic is designed for concurrent operation within the ECS. When the `--all` flag is used, the command leverages the `forEachEntityParallel` iterator. All entity modifications are queued into a thread-local `commandBuffer`. This buffer pattern ensures that writes to the shared entity store are deferred and applied at a safe synchronization point, preventing race conditions.

## API Surface

The public contract is defined by its inheritance from AbstractWorldCommand. The primary entry point is the `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) to O(N) | The command's entry point. Complexity is O(1) for a single target and O(N) for the `--all` flag, where N is the number of NPCs. |

## Integration Patterns

### Standard Usage

This command is intended to be invoked by a server administrator or through automated scripts via the server console or in-game chat.

```
# Advance the targeted NPC by one server tick
/step

# Advance the NPC with a specific UUID by one server tick
/step entity:<uuid>

# Advance all NPCs by a delta-time of 0.5 seconds
/step --all dt:0.5
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new NPCStepCommand()`. The command system manages the lifecycle of this object. Manual instantiation will result in a non-functional object that is not registered to handle any commands.
-   **Assuming Synchronous Execution:** The effects of this command are not immediate. It adds components that are processed by other systems on a subsequent server tick. Do not write code that assumes an NPC's state has changed on the line immediately following the command's invocation.
-   **Directly Calling Execute:** Bypassing the server's command dispatcher to call the `execute` method is dangerous. This circumvents critical infrastructure, including permission checks, context population, and error handling.

## Data Pipeline

The command initiates a state change that flows through several stages of the server's ECS architecture. The process is asynchronous from the perspective of the command issuer.

> Flow:
> Player Input (`/step`) -> Command Parser -> **NPCStepCommand.execute** -> ECS Command Buffer -> ECS Synchronization -> `StepComponent` & `Frozen` added to Entity -> NPC AI System (on next tick) -> Processes `StepComponent` -> NPC State Updated -> `StepComponent` & `Frozen` removed

