---
description: Architectural reference for OpSelfCommand
---

# OpSelfCommand

**Package:** com.hypixel.hytale.server.core.permissions.commands.op
**Type:** Transient Command Object

## Definition
```java
// Signature
public class OpSelfCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The OpSelfCommand class is a concrete implementation of the Command Pattern, designed to encapsulate a single, specific server action: toggling the command executor's own operator status. It functions as a leaf node within the server's command tree, likely registered under a parent command such as *op*.

Its primary architectural role is to serve as a secure bridge between user input and the core **PermissionsModule**. It is not a general-purpose utility; its logic is highly specialized and includes critical safety checks to prevent unauthorized privilege escalation in different server environments (single-player versus multiplayer).

The command's behavior is heavily influenced by global server configuration flags, such as Constants.SINGLEPLAYER and Constants.ALLOWS_SELF_OP_COMMAND, making it an environment-aware component.

## Lifecycle & Ownership
- **Creation:** A single instance of OpSelfCommand is instantiated by the server's command registration system during the bootstrap phase. The system discovers command classes and creates long-lived instances to populate its command map.
- **Scope:** The object instance persists for the entire server session, held as a reference within the central command registry. The execution of its logic, however, is ephemeral and tied to a specific user invocation.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** OpSelfCommand is fundamentally stateless. Its fields are static final Message constants used for localization. All stateful operations, such as reading or modifying a player's permission group, are delegated to the external, stateful **PermissionsModule** singleton.

- **Thread Safety:** This class is not thread-safe and is not designed for concurrent execution. The server's command system MUST ensure that all command executions occur on a single, designated game thread. State modifications are deferred to the PermissionsModule, which is assumed to be internally synchronized and responsible for its own thread safety.

## API Surface
The public contract is defined by its role as a command object. Direct invocation of its methods outside the command system is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| OpSelfCommand() | constructor | O(1) | Creates an instance of the command for registration. |
| canGeneratePermission() | boolean | O(1) | Overrides the parent method to return false, preventing the automatic generation of a permission node for this command. |
| execute(...) | void | O(log N) | Executes the core logic. Complexity is dependent on the backing implementation of the PermissionsModule, which may involve file I/O or map lookups. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is automatically discovered and managed by the server's command framework. A user interacts with it by typing the command in-game. The framework then routes the request to the registered instance.

A conceptual view of its registration might look like this:

```java
// Executed by the server's command registration system at startup
CommandRegistry registry = server.getCommandRegistry();
Command parentCommand = registry.find("op");

// The OpSelfCommand instance is created and registered as a subcommand
parentCommand.registerSubCommand(new OpSelfCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new OpSelfCommand()` in game logic. The command system manages its lifecycle. Creating new instances will have no effect as they will not be registered to receive input.
- **Direct Invocation:** Calling the `execute` method directly bypasses critical infrastructure, including permission checks, command parsing, and context setup. This can lead to unpredictable behavior and server instability.
- **State Reliance:** Do not assume the command holds any state between executions. Each call to `execute` is an independent, atomic operation.

## Data Pipeline
OpSelfCommand acts as a processing node in the server's command handling pipeline. It receives contextual data from the command system and dispatches state change requests to the permissions system.

> Flow:
> Player Input (`/op self`) -> Network Packet -> Server Command Parser -> **OpSelfCommand.execute()** -> PermissionsModule.get() -> Filesystem (permissions.json) -> Confirmation Message -> CommandContext -> Network Packet -> Player Client UI

