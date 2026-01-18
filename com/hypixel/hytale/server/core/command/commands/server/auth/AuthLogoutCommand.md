---
description: Architectural reference for AuthLogoutCommand
---

# AuthLogoutCommand

**Package:** com.hypixel.hytale.server.core.command.commands.server.auth
**Type:** Transient Command

## Definition
```java
// Signature
public class AuthLogoutCommand extends CommandBase {
```

## Architecture & Concepts
The AuthLogoutCommand class is a concrete implementation of the Command Pattern, designed to be discovered and managed by the server's central command system. It encapsulates the logic for a single, user-facing action: terminating the server's authenticated session with Hypixel's backend services.

This class acts as a thin controller, bridging user input from the server console to the core authentication service. Its primary responsibilities are:
1.  **Input Validation:** Performing pre-flight checks to ensure the command is executed in a valid state (e.g., not in single-player mode, server is currently authenticated).
2.  **Delegation:** Offloading the actual logout operation to the ServerAuthManager singleton, which owns the authentication state.
3.  **User Feedback:** Translating the result of the operation (success, failure, or invalid state) into localized, colored messages sent back to the command issuer via the CommandContext.

It intentionally does not contain any authentication logic itself, adhering to the Single Responsibility Principle. The command system guarantees that its execution logic is run on the main server thread to prevent concurrency issues with game state.

### Lifecycle & Ownership
-   **Creation:** A single instance of AuthLogoutCommand is instantiated by the server's command registry during the bootstrap sequence. The system scans for all classes extending CommandBase and registers them for invocation.
-   **Scope:** The object instance is stateless and persists for the entire lifetime of the server process. It is effectively a singleton within the scope of the command registry.
-   **Destruction:** The instance is de-referenced and eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is **stateless**. The static Message fields are immutable constants defined at compile time. All stateful operations are performed on the external ServerAuthManager singleton. Each execution of the command is independent and does not alter the internal state of the AuthLogoutCommand instance.
-   **Thread Safety:** This class is **not thread-safe** and is designed to be operated exclusively by the main server thread. The method name executeSync is a strong convention indicating that it must be called from a synchronized context, which the command dispatch system provides. Unmanaged, multi-threaded access would lead to severe race conditions within the underlying ServerAuthManager.

## API Surface
The public API is minimal and intended for framework use only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AuthLogoutCommand() | Constructor | O(1) | Instantiates the command. Called only by the command system during registration. |
| executeSync(CommandContext) | void | O(1) | Executes the logout logic. This is the command's entry point. Throws NullPointerException if context is null. |

## Integration Patterns

### Standard Usage
A developer or server administrator does not interact with this class directly in code. The standard usage is to invoke the command through the server console or another command-issuing mechanism. The command system handles discovery, instantiation, and invocation.

The following pseudo-code illustrates how the *framework* uses this class:

```java
// Conceptual example of the command dispatcher
CommandRegistry registry = server.getCommandRegistry();
CommandBase command = registry.findCommand("logout"); // Finds the AuthLogoutCommand instance

if (command != null && user.hasPermission(command.getPermission())) {
    CommandContext context = new CommandContext(user, ...);
    // The framework ensures this call happens on the main server thread
    command.executeSync(context);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new AuthLogoutCommand()`. The command system manages the lifecycle of command objects. Manual instantiation creates an unmanaged object that will not be reachable by users.
-   **Manual Invocation:** Avoid calling the `executeSync` method directly. Bypassing the command dispatcher skips critical functionality like permission checks, logging, and thread safety guarantees, which can destabilize the server.

## Data Pipeline
This component operates on a control flow rather than a data transformation pipeline. It receives a command signal and orchestrates actions between other systems.

> Flow:
> Console Input ("logout") -> Command Parser -> **AuthLogoutCommand** -> ServerAuthManager -> CommandContext -> Console Output

