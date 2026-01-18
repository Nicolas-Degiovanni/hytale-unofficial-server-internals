---
description: Architectural reference for PrefabSpawnerSetCommand
---

# PrefabSpawnerSetCommand

**Package:** com.hypixel.hytale.server.core.modules.prefabspawner.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabSpawnerSetCommand extends TargetPrefabSpawnerCommand {
```

## Architecture & Concepts
The PrefabSpawnerSetCommand is a server-side administrative command that provides a direct interface for modifying the configuration of a prefab spawner within a specific world chunk. It is a concrete implementation of the Command Pattern, designed to be discovered and managed by the server's central Command System.

This class acts as the mutation endpoint between a user's intent (expressed as a text command) and the underlying world data. Its direct responsibility is to parse a set of validated arguments and apply them to a target PrefabSpawnerState object. The parent class, TargetPrefabSpawnerCommand, abstracts away the logic of locating the target WorldChunk and its associated PrefabSpawnerState based on world coordinates, allowing this class to focus solely on the *set* operation.

## Lifecycle & Ownership
- **Creation:** A single instance of this command is instantiated by the Command System during server bootstrap. The system scans for command classes and registers them in a central registry for later dispatch.
- **Scope:** The command object itself is a singleton managed by the Command System, persisting for the entire server session. However, its execution is ephemeral; the `execute` method is invoked for a single command request and completes within that scope.
- **Destruction:** The registered instance is discarded when the server shuts down and the Command System is torn down.

## Internal State & Concurrency
- **State:** The class's internal state consists of final fields representing the command's argument definitions (e.g., prefabPathArg, fitHeightmapArg). This configuration is immutable after the constructor is called. The command itself is stateless regarding its execution; all necessary state is passed in via the CommandContext and the target objects.
- **Thread Safety:** This class is **not thread-safe**. The `execute` method mutates shared game state (WorldChunk, PrefabSpawnerState) and must be invoked from the main server thread. The Command System is responsible for ensuring that commands are dispatched and executed in a serialized, thread-safe manner. Concurrent calls to `execute` would lead to race conditions and world data corruption.

## API Surface
The primary contract is the `execute` method, which is invoked by the Command System.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PrefabSpawnerSetCommand() | constructor | O(1) | Defines the command's name ("set"), description, and argument structure for the Command System. |
| execute(context, chunk, prefabSpawner) | void | O(1) | Reads argument values from the CommandContext, applies them to the PrefabSpawnerState, and marks the parent WorldChunk as dirty, ensuring the changes are persisted. |

## Integration Patterns

### Standard Usage
This command is not intended to be used programmatically by other systems. Its interface is the command line, invoked by a server administrator or an automated script. The Command System handles parsing, argument validation, and dispatch.

```
// Example command executed in the server console or by a privileged player.
// This targets the spawner in the chunk at coordinates (10, 20, 30),
// sets its prefab, and configures its behavior.

/prefabspawner set 10 20 30 "hytale:zone1/structures/tower.pfb" fitHeightmap:true inheritSeed:false
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PrefabSpawnerSetCommand()` in your code. The Command System manages the lifecycle of command objects.
- **Manual Execution:** Never call the `execute` method directly. Doing so bypasses critical infrastructure, including permission checks, argument parsing, and context population, which can lead to server instability or invalid state.

## Data Pipeline
This command serves as a control point in the world data pipeline, translating user input into persistent world changes.

> Flow:
> User Input String -> Command System Parser -> **PrefabSpawnerSetCommand** -> Mutates PrefabSpawnerState -> Marks WorldChunk for persistence -> World Save System

