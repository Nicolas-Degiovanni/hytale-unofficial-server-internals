---
description: Architectural reference for WarpReloadCommand
---

# WarpReloadCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.warp
**Type:** Managed Component

## Definition
```java
// Signature
public class WarpReloadCommand extends CommandBase {
```

## Architecture & Concepts
The WarpReloadCommand class is a concrete implementation of the Command Pattern, designed to integrate with the server's core command processing system. It provides a user-facing entry point, typically via a chat command like */warp reload*, for administrators to refresh the server's warp point configuration from its persistent source.

Architecturally, this class serves as a thin bridge between the command system and the core teleportation logic encapsulated within the TeleportPlugin. It does not contain any business logic itself; its sole responsibility is to parse the user's intent, verify permissions, and delegate the actual reload operation to the TeleportPlugin singleton. This separation of concerns ensures that the user interface (the command) is decoupled from the underlying service implementation.

## Lifecycle & Ownership
- **Creation:** An instance of WarpReloadCommand is created and registered by its parent, the TeleportPlugin, during the server's plugin initialization and enabling sequence. It is not intended for on-demand instantiation.
- **Scope:** The object's lifecycle is tightly bound to the TeleportPlugin. It persists in the server's command registry for as long as the plugin is active.
- **Destruction:** The command is de-registered from the command system and becomes eligible for garbage collection when the TeleportPlugin is disabled or during a server shutdown.

## Internal State & Concurrency
- **State:** WarpReloadCommand is fundamentally **stateless**. It contains no mutable instance fields and relies entirely on the state managed by the external TeleportPlugin singleton. All message templates and the logger are static, final, and shared across the application.

- **Thread Safety:** This class is **not thread-safe** and is designed to be executed exclusively on the main server thread. The governing method, **executeSync**, explicitly signals that its execution is synchronized with the primary server game loop.

    **Warning:** Any attempt to invoke **executeSync** from an asynchronous task or a different thread will bypass the server's thread safety guarantees and will almost certainly lead to race conditions, data corruption, or server instability when interacting with the TeleportPlugin's internal state.

## API Surface
The public contract is defined by the CommandBase superclass. The primary entry point is the overridden **executeSync** method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(N) | Delegates to the TeleportPlugin to reload all warp points from storage, where N is the number of warps. Sends success or failure messages to the command issuer via the provided context. Throws unchecked exceptions on catastrophic failure. |

## Integration Patterns

### Standard Usage
This class is not designed for direct programmatic invocation. It is automatically discovered and executed by the server's command handler in response to player input. The standard interaction is through an authorized player executing the command in-game.

```
// Player issues the command in the game client
/warp reload
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create instances using **new WarpReloadCommand()**. The command system manages the lifecycle of registered command objects. Direct instantiation creates an orphan object that is not registered to handle any input.

- **Direct Invocation:** Avoid calling the **executeSync** method directly. This bypasses critical infrastructure, including permission checks, argument parsing, and context provision, which are normally handled by the command processing system. If you need to reload warps programmatically, interact with the service layer directly:
    ```java
    // CORRECT: Use the service directly
    TeleportPlugin.get().loadWarps();

    // INCORRECT: Bypassing the command system
    new WarpReloadCommand().executeSync(someManualContext);
    ```

## Data Pipeline
The data flow for this command is initiated by a user and results in a system state change, with feedback returned to the user.

> Flow:
> Player Command Input -> Server Network Layer -> Command Parser -> **WarpReloadCommand.executeSync** -> TeleportPlugin.loadWarps() -> Filesystem Read -> TeleportPlugin State Update -> CommandContext.sendMessage -> Server Network Layer -> Player Chat UI

