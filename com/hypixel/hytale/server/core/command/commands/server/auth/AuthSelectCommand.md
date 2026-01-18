---
description: Architectural reference for AuthSelectCommand
---

# AuthSelectCommand

**Package:** com.hypixel.hytale.server.core.command.commands.server.auth
**Type:** Transient

## Definition
```java
// Signature
public class AuthSelectCommand extends CommandBase {
```

## Architecture & Concepts
The AuthSelectCommand class is a concrete implementation of the Command Pattern, designed to be discovered and managed by the server's central command system. Its sole responsibility is to provide a user-facing interface for interacting with the `ServerAuthManager`, specifically for selecting a pending authentication profile.

This class acts as a translation layer, converting raw user input from the server console or a player into direct, safe calls against the authentication service. It handles two primary scenarios:
1.  Executing the command with no arguments, which lists the currently available authentication profiles.
2.  Executing the command with an argument (a number or a username), which attempts to select a specific profile via the nested `SelectProfileVariant` class.

Architecturally, it is a terminal component in the command processing chain. It receives a `CommandContext`, performs its logic by delegating to a core system (`ServerAuthManager`), and uses the context to send feedback messages back to the user. It does not emit events or trigger other systems directly.

### Lifecycle & Ownership
-   **Creation:** Instantiated once by the command registration system during server bootstrap. The system likely scans for `CommandBase` implementations and registers them.
-   **Scope:** The instance persists for the entire server session, held as a reference within the command registry.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. It holds no mutable instance fields. All dynamic data, such as the list of pending profiles, is fetched from the `ServerAuthManager` singleton at the moment of execution. The `MESSAGE_*` fields are immutable, static constants.
-   **Thread Safety:** The class is not thread-safe and is not designed to be. The `executeSync` method signature strongly implies that it must be executed on the server's main thread. All interactions with the `ServerAuthManager` singleton are expected to be synchronized by the server's main game loop. Invoking its methods from an external thread will lead to race conditions and undefined behavior.

## API Surface
The primary contract is defined by the `CommandBase` parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(N) | Executes the command. Lists pending profiles if no arguments are given. Complexity is O(N) where N is the number of profiles to list. |
| sendProfileList(context, profiles) | void | O(N) | A static utility method to send a formatted list of game profiles to the command's source. |

## Integration Patterns

### Standard Usage
This class is not intended to be used programmatically. It is invoked by the server's command handler in response to user input.

A user, either in-game or via the server console, would execute the command as follows:

**To list pending profiles:**
```
/auth select
```

**To select a specific profile by index or username:**
```
/auth select 1
/auth select Notch
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not create an instance using `new AuthSelectCommand()`. The object is useless outside the context of the command system, which provides the necessary `CommandContext` for execution.
-   **External Invocation:** Do not call the `executeSync` method directly. This bypasses the command system's parsing, permission checks, and thread synchronization, which can corrupt the state of the `ServerAuthManager`.
-   **Storing State:** Do not attempt to subclass or modify this command to store state. The command's stateless nature is intentional; all authentication state must be managed exclusively by `ServerAuthManager`.

## Data Pipeline
The flow for this command begins with user input and ends with a state change in the authentication manager and feedback to the user.

> Flow:
> User Input (`/auth select 1`) -> Network Layer -> Command Parser -> Command Dispatcher -> **AuthSelectCommand.executeSync** -> ServerAuthManager.selectPendingProfile -> Feedback Message -> User Console

