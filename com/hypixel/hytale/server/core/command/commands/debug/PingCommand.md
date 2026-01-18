---
description: Architectural reference for PingCommand
---

# PingCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug
**Type:** Registered Command Handler

## Definition
```java
// Signature
public class PingCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The PingCommand is a server-side diagnostic tool that provides detailed network latency metrics for a specific player. It is implemented as a concrete command handler within the server's command processing system.

By extending AbstractTargetPlayerCommand, it delegates the complex logic of parsing and resolving a target player from the command's arguments to its parent class. This allows PingCommand to focus solely on its core responsibility: retrieving, formatting, and presenting ping data.

The component's architecture is notable for its use of nested static inner classes (Clear, Graph) to implement sub-commands. This pattern encapsulates related functionality, enhances readability, and promotes code reuse within the command's namespace without polluting the global command registry.

The command retrieves its data from the PacketHandler associated with a target PlayerRef. This data is stored in a HistoricMetric object, indicating that the system does not just track the last ping, but maintains a time-series history of latency measurements over various configurable periods.

## Lifecycle & Ownership
- **Creation:** A single instance of PingCommand is created by the server's command registration service during the bootstrap sequence. The system scans for command implementations and instantiates them for inclusion in a central command registry.
- **Scope:** The PingCommand instance is a long-lived object that persists for the entire lifetime of the server process. It is effectively a stateless service object.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
- **State:** The PingCommand class is fundamentally stateless. It contains no mutable instance fields. All stateful data, such as the `detail` flag's value, is derived from the CommandContext passed into the execute method for each invocation. The latency metrics it processes are owned and managed by the player's PacketHandler, not the command itself.

- **Thread Safety:** This class is inherently thread-safe due to its stateless nature. However, it is designed to be invoked exclusively by the server's main command processing thread. The `execute` method accesses state on the PlayerRef object, which is not guaranteed to be thread-safe for access from other threads.

    **Warning:** Do not invoke the execute method from an asynchronous task or a different thread pool. All interactions with a PlayerRef and its components must be synchronized with the main server tick or follow the engine's prescribed threading model.

## API Surface
The primary entry point is the `execute` method, which is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, source, ref, player, world, store) | void | O(1) | Executes the ping logic. Complexity is constant as it iterates over a fixed number of PongType values and metric periods. |

## Integration Patterns

### Standard Usage
A developer does not typically invoke this class directly. The server's command dispatcher invokes the `execute` method after a player issues the corresponding command in-game. The framework is responsible for constructing the CommandContext and routing the call.

The intended interaction pattern is registration during server startup:
```java
// Conceptual example of command registration
CommandRegistry registry = server.getCommandRegistry();
registry.register(new PingCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PingCommand()` in application logic. The command must be registered with the framework to be accessible. Creating a new instance will result in a non-functional object outside the command system's control.
- **Manual Execution:** Avoid calling the `execute` method directly. The CommandContext object is complex and tightly coupled to the state of the command parser at the moment of execution. Manually constructing this context is unsupported and will lead to unpredictable behavior or NullPointerExceptions.

## Data Pipeline
The flow of data for a typical ping request is initiated by a player and terminates with a formatted message returned to that player.

> Flow:
> Player Chat Input (`/ping ...`) -> Server Command Parser -> **PingCommand** -> PlayerRef.getPacketHandler() -> HistoricMetric -> Message Builder -> CommandContext.sendMessage() -> Player Client UI

