---
description: Architectural reference for AuthLoginCommand
---

# AuthLoginCommand

**Package:** com.hypixel.hytale.server.core.command.commands.server.auth
**Type:** Command Object

## Definition
```java
// Signature
public class AuthLoginCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The AuthLoginCommand class is an implementation of the Composite design pattern, specifically tailored for the server's command system. It functions as a non-executable, structural command that groups related sub-commands under a common namespace.

Its primary role is to act as a routing node within the server's command tree. When a player or the console executes a command beginning with "login", the command system dispatches the request to this object. AuthLoginCommand then delegates the execution to one of its registered children, such as AuthLoginBrowserCommand or AuthLoginDeviceCommand, based on the subsequent arguments.

This class contains no business logic related to authentication. It is purely a structural component for organizing the command-line interface, improving discoverability and usability for server administrators and players.

### Lifecycle & Ownership
-   **Creation:** An instance of AuthLoginCommand is created by the server's central CommandRegistry during the server bootstrap sequence. The registry scans for and instantiates all command classes to build a complete command graph.
-   **Scope:** The object is a stateless component whose instance persists for the entire lifecycle of the server. It is held as a node within the CommandRegistry's internal data structure.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the CommandRegistry is cleared.

## Internal State & Concurrency
-   **State:** The state of this object is effectively immutable after construction. The constructor populates its list of sub-commands, and this collection is not modified during runtime. It holds no session-specific or player-specific data.
-   **Thread Safety:** This class is inherently thread-safe. As its internal state is static post-construction, multiple threads can safely traverse this command node concurrently without locks or synchronization. Responsibility for thread safety during command execution lies with the terminal sub-command (e.g., AuthLoginBrowserCommand), not this collection class.

## API Surface
The public contract is limited to its constructor, as all functional behavior is inherited from AbstractCommandCollection and managed by the command system framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AuthLoginCommand() | Constructor | O(1) | Constructs the command collection node for "login" and registers its child sub-commands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by application-level code. It is a framework component that is discovered and managed by the command system. The only interaction is its instantiation during registration.

```java
// Example from a hypothetical CommandRegistry service
// This is framework-level code, not user code.
CommandRegistry registry = server.getCommandRegistry();
registry.register(new AuthLoginCommand());

// The server's command handler then uses the registry to dispatch commands.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance of AuthLoginCommand in your own code via `new AuthLoginCommand()`. The server's command registry is the sole owner and manager of command objects.
-   **State Manipulation:** Do not attempt to retrieve this object from the registry to add or remove sub-commands at runtime. The command graph is considered immutable after server initialization.

## Data Pipeline
AuthLoginCommand acts as a routing point in the command processing pipeline. It does not transform data but directs the flow based on command arguments.

> Flow:
> Player Input (`/login device ...`) -> Network Layer -> Command Parser -> **AuthLoginCommand** (Routing) -> AuthLoginDeviceCommand (Execution) -> Authentication Service

