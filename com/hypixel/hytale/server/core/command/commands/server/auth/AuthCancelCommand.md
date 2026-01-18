---
description: Architectural reference for AuthCancelCommand
---

# AuthCancelCommand

**Package:** com.hypixel.hytale.server.core.command.commands.server.auth
**Type:** Singleton

## Definition
```java
// Signature
public class AuthCancelCommand extends CommandBase {
```

## Architecture & Concepts
The AuthCancelCommand class is a specific command implementation within the server's Command System. It functions as a user-facing control endpoint for the server's authentication lifecycle, translating a text-based command into a direct programmatic action.

Its primary architectural role is to serve as a bridge between user input (from a console or authorized player) and the core `ServerAuthManager`. It decouples the command parsing and dispatching logic from the authentication business logic, adhering to the Command software design pattern. This class is not a service but rather a stateless handler responsible for a single, discrete action: canceling an in-progress authentication flow.

### Lifecycle & Ownership
- **Creation:** A single instance of AuthCancelCommand is instantiated by the Command System during server bootstrap. It is discovered and registered alongside all other server commands.
- **Scope:** The instance persists for the entire server session. It is held in a central command registry and is not subject to garbage collection until server shutdown.
- **Destruction:** The object is destroyed when the server shuts down and the Command System clears its registry.

## Internal State & Concurrency
- **State:** This class is stateless. The internal `Message` fields are static, final, and immutable. It holds no instance-level data that changes over its lifetime. All necessary state for execution is provided via the `CommandContext` parameter.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. Furthermore, the `executeSync` method signature contractually obligates the Command System to invoke this method on the main server thread. This prevents race conditions related to command execution and ensures that its interaction with the `ServerAuthManager` is serialized with other critical server operations.

## API Surface
The public contract is defined by its parent, `CommandBase`. The primary entry point is `executeSync`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(1) | Executes the command logic. Delegates the cancellation request to the ServerAuthManager and sends a response message to the command issuer via the provided context. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically discovered and executed by the server's command dispatcher in response to user input.

A user or system triggers this command via the server console or an in-game chat command.

**Example User Interaction:**
> `/auth cancel`

The Command System then identifies and invokes the registered `AuthCancelCommand` instance.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new AuthCancelCommand()`. The command system manages the lifecycle of all command objects. Manual instantiation will result in an object that is not registered and will never be called.
- **Manual Invocation:** Do not retrieve this object from the command registry to call `executeSync` directly. This bypasses the dispatcher's setup and context-building logic, which can lead to `NullPointerException` or inconsistent system state. Always dispatch commands through the `CommandSystem` interface.

## Data Pipeline
The flow of data for this command is unidirectional, originating from user input and resulting in a state change within a separate manager and a feedback message.

> Flow:
> User Input (`/auth cancel`) -> Command Parser -> Command Dispatcher -> **AuthCancelCommand.executeSync()** -> ServerAuthManager -> CommandContext -> User Output (Message)

