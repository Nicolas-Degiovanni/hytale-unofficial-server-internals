---
description: Architectural reference for PlayCommandBase
---

# PlayCommandBase

**Package:** com.hypixel.hytale.server.core.modules.singleplayer.commands
**Type:** Template / Base Class

## Definition
```java
// Signature
public abstract class PlayCommandBase extends CommandBase {
```

## Architecture & Concepts
PlayCommandBase is an abstract template class designed to standardize the creation of server commands that manage the network accessibility of a single-player world. It is not a concrete, usable command itself, but rather a blueprint for commands such as `/public`, `/friends`, or `/inviteonly`.

This class serves as a critical bridge between the user-facing Command System and the stateful SingleplayerModule. Its primary architectural role is to encapsulate the complex logic for toggling server access levels. It interprets user input, checks the current server state, and dispatches a formal state change request to the SingleplayerModule.

A core design constraint, enforced via a runtime check against the Constants.SINGLEPLAYER flag, is that any command derived from this base can **only** function in a single-player server environment. This prevents its logic from being erroneously applied in a dedicated multiplayer server context where access control is managed by different systems.

## Lifecycle & Ownership
- **Creation:** This abstract class is never instantiated directly. Concrete subclasses (e.g., a hypothetical PublicCommand) are instantiated once by the server's command registration system during the server bootstrap sequence. The required SingleplayerModule dependency is injected at this time.
- **Scope:** An instance of a concrete subclass persists for the entire server session. It is registered and remains available until the server shuts down.
- **Destruction:** The object is dereferenced and eligible for garbage collection when the server shuts down and its CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** PlayCommandBase is effectively stateless regarding volatile data. It holds immutable configuration provided during construction, namely the reference to the SingleplayerModule and the specific Access type it is designed to manage. During execution, it reads the current server access state from the SingleplayerModule but does not cache or store it locally.
- **Thread Safety:** This class is **not thread-safe**. The `executeSync` method name is a strong indicator that it is designed to be invoked exclusively on the main server thread as part of the game loop's command processing phase. The Command System is responsible for ensuring that command executions are serialized, preventing race conditions. Direct invocation from other threads will lead to undefined behavior and state corruption.

## API Surface
The primary contract is the inherited `executeSync` method. The constructor is protected and intended only for subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(1) | Executes the access toggle logic. Validates single-player context, reads current server state, and requests a state change from the SingleplayerModule. Sends feedback messages to the command source. |

## Integration Patterns

### Standard Usage
A developer must extend PlayCommandBase to create a new, concrete command for managing a specific server access level. The new class is then registered with the server's command system.

```java
// A concrete implementation for managing public server access.
public class PublicCommand extends PlayCommandBase {
    public PublicCommand(@Nonnull SingleplayerModule module) {
        super(
            "public",
            "server.commands.play.public.desc",
            module,
            Access.Public
        );
    }
}

// Conceptual registration during server startup.
// The CommandRegistry would be responsible for this.
SingleplayerModule module = server.getModule(SingleplayerModule.class);
commandRegistry.register(new PublicCommand(module));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to instantiate `new PlayCommandBase(...)`. The class is abstract and cannot be instantiated. You must create a subclass.
- **External Invocation:** Never call the `executeSync` method directly. It must be invoked by the server's Command System, which provides the necessary CommandContext and ensures execution on the correct thread.
- **Misuse for Multiplayer:** Do not extend this class for commands intended for a dedicated multiplayer server. The internal logic is hardcoded for the single-player lifecycle and will fail.

## Data Pipeline
The flow of data and control for a command derived from this class begins with user input and results in a server state change.

> Flow:
> User Input (`/public`) -> Network Packet -> Command Parser -> Command Dispatcher -> **PlayCommandBase.executeSync()** -> SingleplayerModule.requestServerAccess(Access.Public) -> Server Access State Modified -> Confirmation Message to User

---

