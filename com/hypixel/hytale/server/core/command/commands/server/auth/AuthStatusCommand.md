---
description: Architectural reference for AuthStatusCommand
---

# AuthStatusCommand

**Package:** com.hypixel.hytale.server.core.command.commands.server.auth
**Type:** Transient

## Definition
```java
// Signature
public class AuthStatusCommand extends CommandBase {
```

## Architecture & Concepts
The AuthStatusCommand is a concrete implementation of the Command Pattern, designed to provide a diagnostic view into the server's authentication subsystem. It serves as a read-only interface, translating the complex internal state of the **ServerAuthManager** and server **Options** into a human-readable summary for a server administrator.

This class effectively decouples the command issuer (e.g., a console user) from the authentication service. Its primary architectural role is to act as a data aggregator and presenter. It queries multiple sources—runtime authentication state, static server configuration, and token expiry data—and synthesizes them into a coherent status report. This prevents direct, and potentially unsafe, exposure of the **ServerAuthManager** to the command-line interface.

## Lifecycle & Ownership
- **Creation:** A single instance of AuthStatusCommand is created by the server's command registration system during the server bootstrap sequence. It is discovered and instantiated alongside all other server commands.
- **Scope:** The object is a long-lived singleton for the duration of the server's runtime. The command registry maintains a strong reference to the single instance.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
- **State:** AuthStatusCommand is fundamentally stateless. Its fields are static, final **Message** templates, which are immutable. All dynamic information displayed by the command is fetched on-demand from external, stateful systems (**ServerAuthManager**, **Options**) during the execution of the *executeSync* method. It performs no caching.

- **Thread Safety:** This class is thread-safe by design. The `executeSync` method name strongly implies that the command system invokes it from a designated, synchronous game loop or server thread. The command performs only read operations on **ServerAuthManager**, which is expected to be safe for concurrent reads. The command itself introduces no mutable state and therefore no concurrency hazards.

    **WARNING:** Calling `executeSync` from an arbitrary thread outside the server's command dispatcher is unsupported and may lead to race conditions if the underlying authentication systems are not fully thread-safe.

## API Surface
The public contract is minimal, consisting of the constructor for instantiation by the framework and the execution method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AuthStatusCommand() | constructor | O(1) | Initializes the command with its name and description key. Intended for framework use only. |
| executeSync(CommandContext) | void | O(1) | Gathers all authentication-related status information and sends a formatted report to the command context. |

## Integration Patterns

### Standard Usage
This command is not intended to be used programmatically. It is designed to be invoked by a user through the server console or by an in-game entity with sufficient permissions. The command system handles routing and execution.

*Example of user invocation:*
```sh
# A server administrator types this into the server console
auth status
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new AuthStatusCommand()`. The server's command registry is the sole owner and manager of command instances. Direct creation bypasses registration and makes the command non-functional.
- **Programmatic Invocation:** Do not acquire the command instance to call `executeSync` directly. This bypasses critical framework services like permission checking and context setup. If you need to dispatch a command programmatically, use the central command dispatcher service.

## Data Pipeline
The command operates as a terminal stage in a user-initiated data request pipeline. It reads from a core service and formats the output, but does not pass data further downstream.

> Flow:
> User Input (e.g., console) -> Command Dispatcher -> **AuthStatusCommand**.executeSync() -> ServerAuthManager (State Read) -> Message Formatter -> CommandContext (Output Sink) -> User Console

