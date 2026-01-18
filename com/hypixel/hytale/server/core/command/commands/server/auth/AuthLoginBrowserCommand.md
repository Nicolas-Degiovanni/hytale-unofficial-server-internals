---
description: Architectural reference for AuthLoginBrowserCommand
---

# AuthLoginBrowserCommand

**Package:** com.hypixel.hytale.server.core.command.commands.server.auth
**Type:** Transient

## Definition
```java
// Signature
public class AuthLoginBrowserCommand extends CommandBase {
```

## Architecture & Concepts
The AuthLoginBrowserCommand is a user-facing entry point for initiating the server's primary authentication sequence. It functions as an implementation of the Command Pattern, designed to be invoked by a server administrator through the console.

Architecturally, this class serves as a thin bridge between the Command System and the central ServerAuthManager service. Its sole responsibility is to trigger an asynchronous OAuth 2.0 authentication flow. It performs initial state validation (e.g., checking if the server is in single-player mode or already authenticated) before delegating the complex mechanics of the OAuth flow to the ServerAuthManager.

A key design aspect is its asynchronous, non-blocking nature. Upon execution, the command immediately starts the authentication process in the background and returns control to the command system. Feedback regarding the success, failure, or next steps of the authentication process is delivered back to the user's console via callbacks at a later time.

The command also includes a user-experience enhancement: it attempts to automatically open the required authentication URL in the system's default web browser using `java.awt.Desktop`. This is a non-critical feature that gracefully fails on headless systems, where the administrator must manually copy the URL from the console log.

### Lifecycle & Ownership
- **Creation:** A single instance of AuthLoginBrowserCommand is instantiated by the server's command registration system during the server bootstrap phase.
- **Scope:** The command object is a long-lived singleton that persists for the entire duration of the server session. It does not hold per-execution state.
- **Destruction:** The instance is destroyed and garbage collected when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. All state related to authentication (e.g., session tokens, identity tokens, pending profiles) is owned and managed exclusively by the ServerAuthManager singleton. The command's static fields are immutable message templates.

- **Thread Safety:** The `executeSync` method is invoked by the command system, which guarantees execution on a synchronized, main server thread. However, this method initiates an asynchronous operation by calling `startFlowAsync`. The `thenAccept` lambda, which processes the result of the authentication flow, will be executed on a separate thread managed by the Java CompletableFuture framework. All interactions with the CommandContext within this callback, such as `sendMessage`, must be thread-safe as per the command system's contract. The class itself contains no locks or explicit concurrency controls, as it defers all state management to other thread-safe services.

## API Surface
The primary contract is defined by its parent `CommandBase` class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(1) | Triggers the asynchronous OAuth browser authentication flow. This method is non-blocking and returns immediately. All subsequent feedback is delivered to the user via the provided CommandContext. |
| sendPersistenceFeedback(context) | void | O(1) | A package-private static utility that informs the user about the configured credential storage mechanism (e.g., in-memory or persistent). |

## Integration Patterns

### Standard Usage
This command is not intended for programmatic invocation by other systems. It is designed to be executed by a human operator via the server console.

The system's CommandHandler is responsible for parsing the user's input, identifying this command, and invoking it with the appropriate context.

```
// User input in the server console
> auth login browser
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new AuthLoginBrowserCommand()`. The command is useless without being registered in and executed by the command system, which provides the necessary CommandContext for it to function.
- **Synchronous Assumption:** Do not execute this command and immediately assume the server is authenticated. The process is asynchronous and may take several minutes for the user to complete. It can also fail. System logic should rely on state checks via ServerAuthManager, not on the execution of this command.
- **Environment Misuse:** Executing this command on a headless server without log access is an anti-pattern. The primary fallback for authentication is the URL printed to the console log.

## Data Pipeline
The AuthLoginBrowserCommand initiates a control flow rather than a data processing pipeline. The flow involves the command system, the authentication manager, an external identity provider, and finally a callback to deliver the result.

> Flow:
> User Console Input (`auth login browser`) -> Command System Parser -> **AuthLoginBrowserCommand.executeSync** -> ServerAuthManager.startFlowAsync() -> External OAuth Provider
>
> *(User authenticates in browser)*
>
> External OAuth Provider (Redirect) -> Hytale Auth Service -> ServerAuthManager (via polling/callback) -> CompletableFuture resolution -> **AuthLoginBrowserCommand (lambda)** -> CommandContext.sendMessage() -> User Console Output

