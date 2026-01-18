---
description: Architectural reference for ToggleBlockPlacementOverrideCommand
---

# ToggleBlockPlacementOverrideCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player
**Type:** Singleton

## Definition
```java
// Signature
public class ToggleBlockPlacementOverrideCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The ToggleBlockPlacementOverrideCommand is a concrete implementation of the Command design pattern, specifically tailored for server-side execution by a player entity. It inherits from AbstractPlayerCommand, which wires it into the server's command processing framework and ensures that it can only be executed within the context of a valid player session.

Its sole responsibility is to modify a single boolean flag, *overrideBlockPlacementRestrictions*, on a player's Player component. This flag determines if the player can bypass standard rules for placing blocks in the world, a common requirement for administrative or creative modes.

This class acts as a transactional unit of work. It receives all necessary world and entity state via its execute method, performs a direct mutation on the Entity Component System (ECS) store, and provides immediate feedback to the executing player. It does not contain any game logic beyond this state change.

### Lifecycle & Ownership
- **Creation:** A single instance of this class is instantiated by the server's command registration system during server bootstrap. It is discovered, registered by its name and aliases, and held for the server's lifetime.
- **Scope:** The object instance persists for the entire server session. The execute method is invoked transiently each time a player runs the command.
- **Destruction:** The instance is garbage collected when the server shuts down and its command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its internal fields, such as the command name and description, are immutable and set during construction. All stateful operations within the execute method are performed on objects passed in as parameters, primarily the ECS Store.
- **Thread Safety:** The object instance is inherently thread-safe due to its stateless nature. However, the execute method performs mutations on the world state. The Hytale engine guarantees that all command executions are queued and processed on the main server thread, preventing race conditions with other game logic that might access the same Player component.

**WARNING:** Invoking the execute method from an asynchronous thread outside the main server loop will corrupt world state and is strictly forbidden.

## API Surface
The public contract is defined by the abstract parent class. The core logic is contained within the protected execute method, which is the entry point for the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Toggles the block placement override flag on the executing player's component and sends a confirmation message. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. It is invoked automatically by the server's command handler when a player types the command into their console.

The command system is responsible for resolving the player, acquiring the necessary ECS handles, and invoking the command. A conceptual example of the system's internal call flow is shown below.

```java
// Conceptual example of how the Command System invokes this.
// DO NOT replicate this code.
Command command = commandRegistry.find("tbpo");
PlayerRef player = context.getPlayer();
World world = player.getWorld();

// The system provides all necessary state handles
command.execute(context, world.getStore(), player.getRef(), player, world);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ToggleBlockPlacementOverrideCommand()`. The command system manages the lifecycle of all command objects. Creating a new instance serves no purpose and will not be registered.
- **Manual Invocation:** Never call the execute method directly. Doing so bypasses critical infrastructure, including permission checks, context validation, and thread safety guarantees provided by the command processing pipeline.

## Data Pipeline
This component acts as a final endpoint in a user-input-driven data flow. It translates a player's intent into a direct state change within the game world.

> Flow:
> Player Input (`/tbpo`) -> Server Command Parser -> **ToggleBlockPlacementOverrideCommand.execute()** -> ECS Store Mutation -> Player Component Updated

A secondary flow is initiated to provide feedback to the user.

> Flow:
> **ToggleBlockPlacementOverrideCommand.execute()** -> CommandContext.sendMessage() -> Server Message Queue -> Network Packet -> Client Chat UI

