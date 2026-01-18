---
description: Architectural reference for AuthLoginDeviceCommand
---

# AuthLoginDeviceCommand

**Package:** com.hypixel.hytale.server.core.command.commands.server.auth
**Type:** Transient

## Definition
```java
// Signature
public class AuthLoginDeviceCommand extends CommandBase {
```

## Architecture & Concepts
The AuthLoginDeviceCommand is a component of the server's Command System, specifically designed to handle the user-facing initiation of the OAuth 2.0 Device Authorization Grant flow. It acts as the primary bridge between a server administrator's command input (e.g., `/auth login device`) and the underlying authentication services managed by the ServerAuthManager.

Its core responsibility is not to implement the authentication logic itself, but to orchestrate the process:
1.  Perform initial state validation (e.g., check if the server is in single-player mode or if a user is already authenticated).
2.  Trigger the asynchronous device flow within the ServerAuthManager.
3.  Provide immediate feedback to the user that the process has started.
4.  Handle the asynchronous result of the flow (success, failure, or pending profile selection) and translate it into user-friendly messages.

The nested private class, AuthFlow, is a specialized implementation of OAuthDeviceFlow. Its sole purpose is to intercept the device code and verification URL provided by the authentication service and log them to the server console, providing the necessary instructions for the administrator to complete the process in a web browser.

## Lifecycle & Ownership
-   **Creation:** A single instance of AuthLoginDeviceCommand is created by the CommandSystem during server bootstrap. The system scans for all classes extending CommandBase and registers them for runtime invocation.
-   **Scope:** The command object is a long-lived singleton managed by the CommandSystem, persisting for the entire server session. However, the execution of the command via the executeSync method is ephemeral. The asynchronous authentication flow it initiates will outlive the method call, running in a background thread pool.
-   **Destruction:** The object is dereferenced and garbage collected when the server shuts down and the CommandSystem is cleared.

## Internal State & Concurrency
-   **State:** This class is entirely stateless. It contains no mutable instance fields. All state related to the authentication process (session tokens, pending profiles, flow status) is managed externally by the ServerAuthManager singleton. The static Message fields are immutable constants.
-   **Thread Safety:** The class is inherently thread-safe due to its stateless design. The executeSync method is invoked on the main server thread. It safely dispatches a long-running, blocking operation to a background thread pool via ServerAuthManager.startFlowAsync. The result is handled in a thenAccept callback, which is marshaled back to the main thread to ensure safe interaction with the CommandContext for sending messages.

## API Surface
The public API is minimal and intended for use only by the CommandSystem framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AuthLoginDeviceCommand() | constructor | O(1) | Constructs the command, registering its name and description. |
| executeSync(CommandContext) | void | O(1) + async | Validates server state and initiates the device authentication flow. The method returns immediately while the flow proceeds asynchronously. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation. The server's CommandSystem invokes it automatically when a user with appropriate permissions executes the command from the console.

```java
// This is a conceptual example of how the CommandSystem uses the class.
// Do not replicate this pattern in game or plugin logic.

// 1. CommandSystem receives input: "/auth login device"
// 2. It looks up the registered command object.
AbstractCommand command = commandRegistry.get("auth").getSubCommand("login").getSubCommand("device");

// 3. It creates a context and executes the command.
CommandContext context = new CommandContext(/* ... */);
((AuthLoginDeviceCommand) command).executeSync(context);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new AuthLoginDeviceCommand()`. The CommandSystem manages the lifecycle of all command objects. Direct instantiation will result in an un-registered command that is never called.
-   **Manual Invocation:** Avoid calling the executeSync method directly. Doing so bypasses the permission checks, argument parsing, and context setup handled by the CommandSystem, which can lead to unpredictable behavior or security vulnerabilities.
-   **State Manipulation:** Do not attempt to interact with the internal AuthFlow class. It is an implementation detail of the command's orchestration logic. All authentication state should be managed through the public API of ServerAuthManager.

## Data Pipeline
This command initiates a flow that bridges user input, the server console, and external authentication services.

> Flow:
> Console Command -> CommandSystem Parser -> **AuthLoginDeviceCommand.executeSync** -> ServerAuthManager.startFlowAsync -> (Background Thread) OAuth HTTP Requests -> **AuthFlow.onFlowInfo** -> Console Log Output -> (User action in browser) -> (Background Thread) OAuth Polling -> CompletableFuture Resolution -> **thenAccept Callback** -> CommandContext.sendMessage -> Console Feedback Message

