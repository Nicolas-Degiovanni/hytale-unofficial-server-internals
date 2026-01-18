---
description: Architectural reference for WorldMapViewRadiusRemoveCommand
---

# WorldMapViewRadiusRemoveCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.worldmap
**Type:** Transient

## Definition
```java
// Signature
public class WorldMapViewRadiusRemoveCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The WorldMapViewRadiusRemoveCommand is a concrete implementation of the Command Pattern, designed to operate within the server's command processing framework. It is not a standalone service but a specific, stateless handler for a user-initiated action.

Its primary architectural role is to mutate the state of a component attached to a player entity. By extending AbstractTargetPlayerCommand, it delegates the complex and error-prone logic of parsing a target player from command arguments, resolving that player within the entity system, and handling cases where the player is not found. This allows the command to focus exclusively on its core business logic: accessing the target player's WorldMapTracker component and resetting a specific property.

This class acts as a terminal node in the command dispatch tree, translating a text-based server command into a direct state change within the game's Entity Component System (ECS).

### Lifecycle & Ownership
- **Creation:** A single instance is created by the command registration service during server bootstrap. It is registered as a subcommand, likely under a path such as *worldmap viewradius remove*.
- **Scope:** The object instance is effectively a singleton that persists for the entire server session. However, its operation is transient; it holds no state between executions.
- **Destruction:** The instance is discarded when the server shuts down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. Its configuration (name and description) is provided via the constructor and does not change. All data required for an operation is passed as arguments to the execute method.
- **Thread Safety:** The instance itself is inherently thread-safe due to its stateless nature. However, the execute method performs mutations on shared game state (the EntityStore). The command system framework is responsible for ensuring that command execution is synchronized with the main server thread to prevent race conditions and maintain data consistency within the ECS. Direct invocation from an asynchronous context is unsafe.

## API Surface
The primary contract is the protected execute method, which is invoked by the parent AbstractTargetPlayerCommand after it successfully resolves a target player.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(1) | Resets the view radius override on the target player's WorldMapTracker. Sends a confirmation or error message to the command source. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is used exclusively by the server's command processing system. A server administrator triggers its execution by typing the corresponding command in-game or via the server console.

```text
// User-facing command execution
/worldmap viewradius remove PlayerName
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WorldMapViewRadiusRemoveCommand()`. The instance is managed by the command registry. Creating a new instance serves no purpose as it is not registered to handle any command strings.
- **Manual Execution:** Avoid calling the execute method directly. Doing so bypasses the critical player-lookup logic, permission checks, and argument validation provided by the AbstractTargetPlayerCommand superclass and the broader command framework.

## Data Pipeline
This component acts on a control flow, translating a user command into a state change and a feedback message.

> Flow:
> User Input String -> Command Parser -> **WorldMapViewRadiusRemoveCommand** -> EntityStore Access -> WorldMapTracker State Mutation -> CommandContext -> Feedback Message -> Network Packet -> Client UI

