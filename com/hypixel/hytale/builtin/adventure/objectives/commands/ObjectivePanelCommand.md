---
description: Architectural reference for ObjectivePanelCommand
---

# ObjectivePanelCommand

**Package:** com.hypixel.hytale.builtin.adventure.objectives.commands
**Type:** Singleton

## Definition
```java
// Signature
public class ObjectivePanelCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The ObjectivePanelCommand is a server-side command handler responsible for bridging player input to a specific UI action. As a concrete implementation of the Command Pattern, its sole function is to open the administrative objectives panel for the player who executes the command.

This class extends AbstractPlayerCommand, which is a critical architectural constraint. This base class ensures that the command can only be executed by an entity with a Player component, automatically providing the necessary player-specific context (PlayerRef) to the execution logic. It acts as a terminal node in the command processing system, translating a parsed command directly into a call to the UI PageManager system. It does not manage state or orchestrate complex workflows; it is a simple, stateless trigger.

## Lifecycle & Ownership
- **Creation:** A single instance of ObjectivePanelCommand is created by the server's CommandSystem during the bootstrap phase. The system scans for command definitions and registers them in a central registry for the server's lifetime.
- **Scope:** The object is a server-scoped singleton. It persists for the entire duration of a server session.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server is shutting down and the CommandSystem clears its registry.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. Its internal fields for the command name and permission key are set in the constructor and never change. All operations in the execute method are performed on context objects passed as parameters.
- **Thread Safety:** The class is inherently **thread-safe**. Since it holds no mutable state, concurrent invocations cannot interfere with each other. In practice, the server's command execution logic ensures that commands are processed sequentially on the main server thread, preventing any concurrency issues with the game state objects it accesses.

## API Surface
The primary contract is the overridden execute method. Direct invocation of the constructor is not part of the public API.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Retrieves the Player component for the executing player and instructs their PageManager to open the ObjectiveAdminPanelPage. This is a server-authoritative action that results in a network packet being sent to the client. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly. It is automatically discovered and invoked by the server's command processing system in response to a player typing the registered command into chat. A developer would typically only interact with this class to register it with the system.

For programmatic execution, one would use the central command system to dispatch the command on behalf of a player, which correctly handles permissions and context setup.

```java
// Hypothetical example of programmatic dispatch
// Do NOT call the execute method directly.
CommandSystem commandSystem = server.getCommandSystem();
PlayerRef targetPlayer = ...;
commandSystem.dispatch(targetPlayer, "objective panel");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ObjectivePanelCommand()`. The command system manages the lifecycle of command objects. Creating a new instance serves no purpose as it will not be registered to handle any input.
- **Direct Invocation:** Never call the `execute` method directly. Doing so bypasses the entire command processing pipeline, including critical permission checks, argument parsing, and context validation. This can lead to server instability or security vulnerabilities.

## Data Pipeline
The flow of data for this command begins with the client and ends with a UI update on the same client, orchestrated by the server.

> Flow:
> Client Chat Packet -> Server Network Layer -> Command Parser -> **ObjectivePanelCommand** -> Player PageManager -> Server Network Layer -> Client UI Update Packet

