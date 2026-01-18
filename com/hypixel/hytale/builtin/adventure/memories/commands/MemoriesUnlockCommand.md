---
description: Architectural reference for MemoriesUnlockCommand
---

# MemoriesUnlockCommand

**Package:** com.hypixel.hytale.builtin.adventure.memories.commands
**Type:** Transient

## Definition
```java
// Signature
public class MemoriesUnlockCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The MemoriesUnlockCommand class is a server-side administrative command that provides a direct interface for unlocking all adventure mode "memories". Architecturally, it serves as a thin bridge between the server's command processing system and the business logic encapsulated within the MemoriesPlugin.

By extending AbstractWorldCommand, this class automatically integrates with the server's command registry and argument parser. This inheritance mandates an implementation of the execute method, which is supplied with the necessary World and CommandContext, ensuring that the command operates within a valid and secure server state. Its primary role is to expose a specific, high-privilege function of the MemoriesPlugin to server operators or automated scripts via the in-game console.

## Lifecycle & Ownership
- **Creation:** An instance of MemoriesUnlockCommand is created by the server's command system during the plugin loading phase. The system scans the plugin's components, discovers this command, and registers it for use.
- **Scope:** The registered command instance persists for the entire lifecycle of the server or until the parent MemoriesPlugin is unloaded. It is effectively a singleton managed by the command registry.
- **Destruction:** The instance is deregistered and becomes eligible for garbage collection when the server shuts down or the MemoriesPlugin is disabled, at which point the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is stateless. Its only field is a static final Message object, which is immutable. No instance-level state is stored, ensuring that each execution is independent and idempotent from the perspective of the command class itself.
- **Thread Safety:** The class is inherently thread-safe due to its stateless design. However, the overall operation's thread safety is delegated to the downstream MemoriesPlugin.get().recordAllMemories() call. The command executor must assume that the plugin handles its own concurrency and state management correctly. No locks or synchronization primitives are used within this class.

## API Surface
The public contract is defined by its role as a command and is limited to the inherited execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | Variable | Executes the command logic. Complexity is delegated to MemoriesPlugin.recordAllMemories(). This method is not intended for direct invocation. |

## Integration Patterns

### Standard Usage
This class is not designed to be used programmatically. It is invoked by an authorized entity (e.g., a server administrator) through the server console or in-game chat.

```plaintext
// Example invocation from the server console
/memories unlockAll
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new MemoriesUnlockCommand()`. The command system manages its lifecycle. Manually calling the execute method bypasses critical infrastructure such as permission checks, context validation, and error handling provided by the command system.
- **Programmatic Invocation:** Do not treat this class as a general-purpose API for unlocking memories from other code. If you need to trigger this logic from another system, you should interface directly with the MemoriesPlugin, which owns the core functionality. This class is strictly a command-line adapter.

## Data Pipeline
The flow of data and control for this command is linear and unidirectional. It receives a trigger from the command system and initiates a state change in a separate plugin, then provides feedback.

> **Execution Flow:**
> Server Console Input (`/memories unlockAll`) -> Command Parser -> **MemoriesUnlockCommand.execute()** -> MemoriesPlugin.recordAllMemories() -> World State Mutation

> **Feedback Flow:**
> **MemoriesUnlockCommand.execute()** -> CommandContext.sendMessage() -> Network Subsystem -> Client Chat UI

