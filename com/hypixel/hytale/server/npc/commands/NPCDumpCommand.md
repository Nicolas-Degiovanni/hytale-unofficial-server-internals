---
description: Architectural reference for NPCDumpCommand
---

# NPCDumpCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Transient

## Definition
```java
// Signature
public class NPCDumpCommand extends NPCWorldCommandBase {
```

## Architecture & Concepts
The NPCDumpCommand is a server-side diagnostic tool designed for introspection of an NPC's internal state. It functions as a read-only command, providing developers and server administrators a snapshot of the component hierarchy that defines an NPC's behavior through its assigned Role.

This command is a terminal node in the server's command processing system. When invoked, it targets a selected NPCEntity and recursively traverses its component tree, starting from the root Role. The primary architectural contribution of this class is to provide a bridge between the complex, in-memory object graph of an NPC's components and a serializable, human-readable representation.

It supports two distinct output formats, controlled by a command-line flag:
1.  **Text Format:** A simple, indented list for quick inspection in the console.
2.  **JSON Format:** A structured, machine-readable format suitable for analysis, scripting, or feeding into other diagnostic tools.

The traversal logic relies on the IAnnotatedComponent and IAnnotatedComponentCollection interfaces, allowing it to generically walk any component structure without needing specific knowledge of each component's implementation.

## Lifecycle & Ownership
- **Creation:** A single instance of NPCDumpCommand is created by the server's command registration system during the server bootstrap phase. It is not instantiated per-execution.
- **Scope:** The singleton instance persists for the entire server session, held within a central command registry or dispatcher.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless regarding its execution. Its only instance field, jsonArg, is initialized in the constructor and is immutable thereafter. All state required for execution (the target NPC, the world, the command context) is passed as arguments to the execute method.
- **Thread Safety:** **Not Thread-Safe.** This command is designed to be executed exclusively on the main server thread (the world tick thread). It reads directly from live game state (NPCEntity, Role, World), which is not thread-safe. The server's command execution model guarantees single-threaded access, preventing concurrency issues.

**WARNING:** Invoking the execute method from any thread other than the main server thread will lead to race conditions, data corruption, and server instability.

## API Surface
The public contract is fulfilled by inheriting from the command system's base classes. The primary entry point is the overridden execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, npc, world, store, ref) | void | O(N) | Executes the dump operation. N is the number of components in the NPC's Role. Traverses the component hierarchy and logs the structure to the server console. |

## Integration Patterns

### Standard Usage
This command is not intended to be used programmatically. It is invoked by a user with appropriate permissions through the server console. The command requires a pre-selected NPC.

```sh
# Select an NPC first, then run the dump command for text output
/npc select <npc_id>
/npc dump

# To get structured JSON output, use the --json flag
/npc select <npc_id>
/npc dump --json
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new NPCDumpCommand()`. The command system manages the lifecycle of command objects. Direct instantiation creates an unmanaged object that is not registered to handle console input.
- **Manual Invocation:** Do not call the `execute` method directly. Bypassing the command system's dispatcher prevents critical operations such as permission checking, argument parsing, and context setup, which can lead to NullPointerExceptions and invalid state.

## Data Pipeline
The command transforms in-memory game state into a textual representation for logging. It does not modify any game state.

> Flow:
> Server Console Input -> Command Dispatcher -> **NPCDumpCommand**.execute -> Traverses NPCEntity.Role -> Aggregates data into StringBuilder or JsonObject -> NPCPlugin Logger -> Server Log Output

