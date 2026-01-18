---
description: Architectural reference for PrefabSpawnerWeightCommand
---

# PrefabSpawnerWeightCommand

**Package:** com.hypixel.hytale.server.core.modules.prefabspawner.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabSpawnerWeightCommand extends TargetPrefabSpawnerCommand {
```

## Architecture & Concepts
The PrefabSpawnerWeightCommand is a server-side administrative command handler responsible for modifying the procedural generation parameters of a specific world chunk at runtime. It functions as a direct interface for developers and server operators to tune the spawn probabilities of prefabs managed by a PrefabSpawnerState component.

Architecturally, this class is a terminal node in the server's Command System. It does not contain game logic itself; rather, it acts as a transactional script that is invoked by the command parser. Its primary role is to translate user input into a specific, stateful change on a world data component (`PrefabSpawnerState`).

By inheriting from TargetPrefabSpawnerCommand, it delegates the complex and error-prone logic of resolving a command's execution context to a specific `WorldChunk` and its associated `PrefabSpawnerState`. This promotes a clean separation of concerns, allowing this class to focus solely on the mutation of prefab weight data.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by the server's central Command System when a matching command string is parsed from a user (e.g., via the server console). It is never created directly by core game systems.
- **Scope:** The object's lifetime is exceptionally short, scoped exclusively to the execution of a single command. It is created, its `execute` method is called once, and it is then immediately dereferenced.
- **Destruction:** The instance becomes eligible for garbage collection as soon as the `execute` method returns. The Command System does not maintain any long-term references to command handler instances.

## Internal State & Concurrency
- **State:** This class is effectively stateless. The `prefabArg` and `weightArg` fields are final definitions of the command's argument structure, not mutable instance data. All state modifications are performed on external objects (`PrefabSpawnerState`, `WorldChunk`) passed into the `execute` method.
- **Thread Safety:** This class is not thread-safe and is not designed to be. The server's Command System guarantees that all command execution occurs on a single, designated server thread, preventing race conditions. Any direct invocation from an asynchronous task or an alternate thread is a severe violation of the engine's threading model and will lead to world data corruption. The subsequent call to `chunk.markNeedsSaving` delegates concurrency concerns for world persistence to the WorldChunk and its backing systems.

## API Surface
The public contract is fulfilled by overriding the `execute` method from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, chunk, prefabSpawner) | void | O(1) | Mutates the PrefabWeights of the target spawner. A negative weight is treated as a removal operation. Marks the parent chunk as dirty, queuing it for persistence. |

## Integration Patterns

### Standard Usage
This class is never used directly in code. It is invoked implicitly by the server's command dispatch mechanism. A developer or administrator interacts with it via text commands in the server console.

A hypothetical invocation from within the command system might look like this:

```java
// This is a conceptual example of how the CommandSystem dispatches the command.
// Developers do NOT write this code.

CommandContext context = buildContextForUserInput("/prefabspawner weight hytale:dungeon_1 50.0");
PrefabSpawnerWeightCommand command = new PrefabSpawnerWeightCommand();

// The parent class's logic would resolve the target chunk and spawner
WorldChunk targetChunk = findChunkFromContext(context);
PrefabSpawnerState targetSpawner = getSpawnerFromChunk(targetChunk);

// The command is executed with the resolved targets
command.execute(context, targetChunk, targetSpawner);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new PrefabSpawnerWeightCommand()` in game logic. The class requires a fully populated `CommandContext` from the command framework to function. Bypassing the framework will result in NullPointerExceptions and undefined behavior.
- **External Execution:** Do not acquire an instance of this class and call its `execute` method from other systems. This breaks the command pattern, bypasses permission checks, and violates the single-threaded contract for world modification.
- **Stateful Implementation:** Do not add non-final fields to this class to carry state. The Command System may reuse class definitions but provides no guarantees about instance lifecycle beyond a single execution.

## Data Pipeline
The flow of data for a weight modification command is unidirectional, originating from user input and terminating in a persistent world state change.

> Flow:
> Console Input -> Command System Parser -> Argument Binder -> **PrefabSpawnerWeightCommand.execute()** -> PrefabSpawnerState.setPrefabWeights() -> WorldChunk.markNeedsSaving() -> World Persistence Thread

