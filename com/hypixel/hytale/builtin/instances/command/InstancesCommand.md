---
description: Architectural reference for InstancesCommand
---

# InstancesCommand

**Package:** com.hypixel.hytale.builtin.instances.command
**Type:** Command Handler

## Definition
```java
// Signature
public class InstancesCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The InstancesCommand class serves as the primary, user-facing entry point for all server-side world instancing operations. It is not a service that performs business logic, but rather a command dispatcher that forms the root of a command tree. Its core architectural function is to parse player-entered commands related to "instances" and delegate the request to a more specialized subcommand or, in its default case, to a user interface.

When a player executes the base command, for example `/instances`, this class does not modify game state directly. Instead, it acts as a bridge between the server's Command System and the player's UI System. It retrieves the player's context and instructs their client to open the InstanceListPage, a dedicated UI for managing world instances.

This class and its nested subclasses, like InstancesEditCommand, exemplify a **Composite Pattern** for command handling. The top-level command is a collection that holds other commands, allowing for a clean, hierarchical command structure (e.g., `/instances edit new`).

## Lifecycle & Ownership
- **Creation:** A single instance of InstancesCommand is created by the server's CommandSystem during the bootstrap phase. The system scans for and registers all available command handlers at startup.
- **Scope:** The registered instance is a stateless handler that persists for the entire lifecycle of the server. It does not hold any per-player or per-request state.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the CommandSystem is dismantled.

## Internal State & Concurrency
- **State:** This class is effectively **immutable** after construction. Its state, which consists of its name, aliases, and a collection of subcommands, is configured entirely within the constructor and is not modified during runtime.
- **Thread Safety:** The object instance is inherently thread-safe due to its stateless design. The execute method is designed to be called by the server's command processing thread, which is synchronized with the main game loop. The parameters passed into the execute method, such as the Store and World, are managed by the engine's concurrency model. **WARNING:** Developers should not invoke the execute method from arbitrary threads.

## API Surface
The public contract is defined by its role as a command handler, with the execute method being the sole entry point for the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Retrieves the executing player's component and opens the InstanceListPage UI. This is the default action for the base `/instances` command. |

## Integration Patterns

### Standard Usage
This class is not intended to be used like a traditional service. It is registered with the CommandSystem, which then invokes it based on player input. A developer's interaction is typically limited to ensuring it is registered.

```java
// Example of how the CommandSystem registers this handler during server startup
CommandSystem commandSystem = server.getCore().getCommandSystem();
commandSystem.register(new InstancesCommand());
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Never call the execute method directly in your code. Doing so bypasses critical infrastructure, including permission checks, context setup, and command parsing provided by the CommandSystem. This can lead to inconsistent state or server instability.
- **Post-Construction Modification:** Do not attempt to add aliases or subcommands to the instance after it has been constructed. The command hierarchy is expected to be static throughout the server's lifetime.

## Data Pipeline
The execution of this command initiates a flow that transitions from a server-side command context to a client-side UI interaction.

> Flow:
> Player Chat Input (`/instances`) -> Network Packet -> Server Command Parser -> CommandSystem Dispatcher -> **InstancesCommand.execute()** -> Player.getPageManager().openCustomPage() -> UI System Event -> Client Renders InstanceListPage

---
# InstancesCommand.InstancesEditCommand

**Package:** com.hypixel.hytale.builtin.instances.command
**Type:** Command Handler

## Definition
```java
// Signature
public static class InstancesEditCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
InstancesEditCommand is a nested static class that functions as a subcommand collection within the main InstancesCommand. It does not contain any execution logic itself. Its sole purpose is to group related instance-editing commands under a common namespace, `/instances edit`.

This class follows the same **Composite Pattern** as its parent, acting as a non-terminal node in the command tree. It holds references to leaf commands like InstanceEditNewCommand and InstanceEditCopyCommand, which contain the actual logic for modifying instance data. This design promotes separation of concerns and keeps the command structure organized and extensible.

## Lifecycle & Ownership
- **Creation:** Instantiated by the parent InstancesCommand constructor. Its lifecycle is directly tied to its parent.
- **Scope:** Persists for the entire server session as part of the parent InstancesCommand object.
- **Destruction:** Destroyed when the parent InstancesCommand is garbage collected during server shutdown.

## Internal State & Concurrency
- **State:** **Immutable**. Like its parent, its collection of subcommands is defined during construction and never changes. It is a stateless container.
- **Thread Safety:** Inherently thread-safe. It has no executable method and simply serves as a structural component in the command registry.

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. It is an internal implementation detail of the InstancesCommand, used to build the command hierarchy. The system interacts with it via the CommandSystem's parsing logic.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** There is no reason to create an instance of InstancesEditCommand outside of the parent InstancesCommand constructor. The command system will not recognize a standalone instance.
- **Direct Invocation:** This class extends AbstractCommandCollection and does not have an `execute` method to invoke. Attempting to treat it as an executable command will result in an error.

