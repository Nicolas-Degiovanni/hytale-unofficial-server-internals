---
description: Architectural reference for PrefabSpawnerGetCommand
---

# PrefabSpawnerGetCommand

**Package:** com.hypixel.hytale.server.core.modules.prefabspawner.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabSpawnerGetCommand extends TargetPrefabSpawnerCommand {
```

## Architecture & Concepts
The PrefabSpawnerGetCommand is a server-side administrative command that provides a read-only view into the configuration of a specific prefab spawner. It serves as a diagnostic and introspection tool for server operators to debug world generation or verify the state of a spawner entity within a given WorldChunk.

This class is a concrete implementation within the server's Command System. It operates as a leaf node in the command execution chain, responsible for retrieving data from a PrefabSpawnerState component and formatting it for display to the user who invoked the command. Its parent, TargetPrefabSpawnerCommand, abstracts the logic of locating the target chunk and the associated spawner state, allowing this class to focus solely on data presentation.

## Lifecycle & Ownership
- **Creation:** A single instance is created and registered by the server's command management system during the server bootstrap phase. It is not intended to be instantiated multiple times.
- **Scope:** The command object is a long-lived singleton that persists for the entire server session. The parameters passed to its execute method, such as CommandContext, are transient and scoped to a single command invocation.
- **Destruction:** The instance is dereferenced and garbage collected during server shutdown when the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is stateless. It contains no mutable fields and all required data is provided as arguments to the execute method. Its behavior is entirely determined by its inputs.
- **Thread Safety:** The class is inherently thread-safe due to its stateless design. However, command execution is expected to be serialized and performed on the main server thread to guarantee safe access to the underlying WorldChunk and PrefabSpawnerState data, which are not thread-safe for concurrent modification.

## API Surface
The public contract is defined by the parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, chunk, prefabSpawner) | protected void | O(1) | Reads properties from the provided PrefabSpawnerState and sends formatted messages to the command issuer via the CommandContext. |

## Integration Patterns

### Standard Usage
This command is designed to be invoked by a server administrator through the server console or in-game chat. The command system handles parsing, routing, and execution.

```java
// This code is conceptual and represents the server's internal dispatching.
// Developers do not call this directly.

// User types: /prefabspawner get
CommandDispatcher dispatcher = server.getCommandDispatcher();
dispatcher.process("prefabspawner get", commandIssuer);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class directly. The command system manages its lifecycle. `new PrefabSpawnerGetCommand()` will result in an unmanaged object that is not registered to handle any commands.
- **Programmatic Invocation:** This command should not be called from other game systems or plugins. It is a user-facing diagnostic tool. For programmatic access to spawner state, interact with the PrefabSpawnerState component directly via the entity-component system.

## Data Pipeline
The command acts as a terminal stage in a user-initiated data request pipeline. It translates internal game state into human-readable messages.

> Flow:
> User Input (e.g., "/ps get") -> Command Dispatcher -> **PrefabSpawnerGetCommand** -> Read PrefabSpawnerState -> Format Message Objects -> CommandContext -> Network Layer -> Client Chat UI

