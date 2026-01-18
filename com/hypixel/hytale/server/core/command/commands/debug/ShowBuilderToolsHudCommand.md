---
description: Architectural reference for ShowBuilderToolsHudCommand
---

# ShowBuilderToolsHudCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug
**Type:** Transient

## Definition
```java
// Signature
public class ShowBuilderToolsHudCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The ShowBuilderToolsHudCommand is a server-side command implementation responsible for controlling the visibility of specific creative mode user interface elements. As a subclass of AbstractPlayerCommand, it is designed to be executed within the context of a specific player entity, not globally.

Its primary role is to act as a bridge between a player's chat input and the state of their personal HudManager component. The command parses a simple flag argument (--hide) to determine whether to show or hide the builder tools legend. This provides a direct, user-driven mechanism for customizing the in-game HUD for building-focused tasks, typically in a creative or debug game mode.

This class is a terminal node in the command processing system; it contains the final business logic for a specific server action and does not delegate to other systems beyond the player's own components.

### Lifecycle & Ownership
- **Creation:** A single instance of this class is instantiated by the server's command registration system during the server bootstrap phase. It is registered against the command name "builderToolsLegend".
- **Scope:** The object instance is a long-lived singleton that persists for the entire server session. The state of the command object itself does not change between executions.
- **Destruction:** The instance is dereferenced and garbage collected when the server shuts down or the command is explicitly unregistered from the command system.

## Internal State & Concurrency
- **State:** This class is effectively stateless regarding its execution. Its only member, hideArg, is a configuration object initialized in the constructor and is immutable thereafter. All state required for execution (the player, the world, the entity store) is passed as arguments to the execute method.
- **Thread Safety:** **This class is not thread-safe.** Command execution is expected to occur exclusively on the main server thread that manages the game world and its entities. The execute method directly mutates the state of a Player component via its HudManager, an operation that is not safe to perform from any other thread.

## API Surface
The public API is defined by its parent, AbstractPlayerCommand. The core interaction point is the overridden execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Executes the command logic. Retrieves the player's HudManager and toggles the visibility of BuilderToolsLegend and BuilderToolsMaterialSlotSelector based on the presence of the hide flag. |

## Integration Patterns

### Standard Usage
This command is intended to be invoked by a player through the in-game chat console. The server's command dispatcher resolves the command name, verifies permissions, and invokes the execute method with the appropriate context.

```java
// This code is conceptual and represents the server's internal dispatching.
// A developer would not write this; a player triggers it via chat.

// Player types: /builderToolsLegend --hide
CommandContext context = createFromPlayerInput("/builderToolsLegend --hide");
Command command = commandRegistry.get("builderToolsLegend");

// The dispatcher invokes the command
command.execute(context, ...);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ShowBuilderToolsHudCommand()`. The command must be retrieved from the server's central command registry to ensure proper lifecycle management and registration.
- **Manual Invocation:** Avoid calling the execute method directly. Bypassing the command dispatcher skips critical steps like permission checks and context setup, which can lead to server instability or security vulnerabilities.
- **Asynchronous Execution:** Never invoke this command from an asynchronous task or a different thread. Modifying player component state must be synchronized with the main server game loop.

## Data Pipeline
The data flow for this command originates from the client and results in a state change that is then synchronized back to the client.

> Flow:
> Player Chat Input (`/builderToolsLegend`) -> Client-to-Server Network Packet -> Server Command Parser -> **ShowBuilderToolsHudCommand.execute()** -> Player.HudManager State Mutation -> Server-to-Client Network Packet (HUD Update) -> Client Renders Updated HUD

