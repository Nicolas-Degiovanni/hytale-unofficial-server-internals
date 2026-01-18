---
description: Architectural reference for AuthCommand
---

# AuthCommand

**Package:** com.hypixel.hytale.server.core.command.commands.server.auth
**Type:** Transient

## Definition
```java
// Signature
public class AuthCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The AuthCommand class is a **Command Aggregator** that serves as the top-level entry point for all server authentication commands. It embodies the Composite design pattern, grouping a collection of related, more specific command objects (AuthStatusCommand, AuthLoginCommand, etc.) under a single, user-facing namespace: *auth*.

This class contains no execution logic itself. Its sole responsibility is to act as a structural container and router within the server's command processing system. When a user or system issues a command like `/auth login <credentials>`, the central CommandRegistry first routes the request to this AuthCommand instance based on the "auth" token. This class then delegates the remainder of the command to the appropriate registered sub-command handler, in this case, AuthLoginCommand.

This pattern decouples the main command router from the specific implementations of each sub-command, promoting modularity and making it simple to add, remove, or modify authentication-related commands without altering the core command system.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the server's central CommandRegistry during the bootstrap phase. The registry typically discovers command classes via reflection or explicit registration and creates a singleton instance for its internal mapping.
- **Scope:** The object's lifecycle is bound to the CommandRegistry. It persists for the entire duration of the server session.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** Effectively **Immutable** after construction. The internal list of sub-commands is populated exclusively within the constructor and is not designed to be modified at runtime. This class holds no other dynamic state.
- **Thread Safety:** This class is inherently **Thread-Safe**. Its immutable-after-construction nature ensures that it can be safely accessed by multiple player command-processing threads concurrently without locks or synchronization primitives. The responsibility for thread-safe execution of the actual logic lies with the individual sub-command classes it contains.

## API Surface
The primary public contract of this class is its constructor, which is invoked by the command system during registration. All other interactions are handled through the AbstractCommandCollection interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AuthCommand() | constructor | O(1) | Constructs the command collection and registers all authentication-related sub-commands. This is the primary integration point for the command system. |

## Integration Patterns

### Standard Usage
A developer does not typically interact with this class directly. The server's command system discovers and manages it. The following example illustrates how the system might register this command collection.

```java
// Example of how the CommandRegistry might initialize this command
CommandRegistry registry = server.getCommandRegistry();
registry.register(new AuthCommand());

// Users then interact with it via text commands
// /auth status
// /auth login <user> <pass>
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid creating instances of AuthCommand in general application logic. It should only be instantiated once for registration within the server's core command system. Creating multiple instances serves no purpose and wastes memory.
- **Runtime Modification:** Do not attempt to retrieve this object and modify its sub-command list after initialization. The command system relies on a static, predictable command structure. Dynamically altering the collection can lead to race conditions and undefined behavior.

## Data Pipeline
AuthCommand acts as an initial routing step in the command processing pipeline. It validates the primary command token and delegates to a more specific handler.

> Flow:
> User Input (`/auth login ...`) -> Network Layer -> Command Parser -> CommandRegistry -> **AuthCommand** (Router) -> AuthLoginCommand (Executor) -> Authentication Service -> Player Session Update

