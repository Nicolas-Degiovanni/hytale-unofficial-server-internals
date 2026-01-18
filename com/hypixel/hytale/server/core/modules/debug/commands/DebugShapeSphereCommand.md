---
description: Architectural reference for DebugShapeSphereCommand
---

# DebugShapeSphereCommand

**Package:** com.hypixel.hytale.server.core.modules.debug.commands
**Type:** Command Object

## Definition
```java
// Signature
public class DebugShapeSphereCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The DebugShapeSphereCommand is a concrete implementation of the **Command Pattern**, designed to integrate with the server's command processing framework. It functions as a terminal command handler, responsible for executing a specific debugging action when invoked by a player.

Architecturally, this class serves as a bridge between the high-level command system and the low-level debugging visualization utilities. Its primary responsibility is to parse the context of a command invocation—specifically, the player who executed it—and translate that into an action within the game world. It achieves this by interacting with two core systems:

1.  **Entity Component System (ECS):** It queries the ECS via the provided Store and Ref to retrieve the player's `TransformComponent`, which contains their current world position.
2.  **Debug Rendering System:** It uses the static `DebugUtils` class to inject a new debug shape (a sphere) into the world's rendering state.

This command is intentionally simple and serves as a leaf node in a larger command tree, likely registered under a parent command such as *debug* or *shape*.

## Lifecycle & Ownership
-   **Creation:** Instances of DebugShapeSphereCommand are not created on-demand. They are instantiated once by the server's command registration system during the server bootstrap or module loading phase. The system discovers and registers all classes that extend `AbstractPlayerCommand` or a similar base.
-   **Scope:** A single instance of this class persists for the entire server session. It is effectively a singleton managed by the command framework. The object itself is stateless; all relevant data is passed into the `execute` method during each invocation.
-   **Destruction:** The object is garbage collected only when the server shuts down or if its module is unloaded, at which point the command registry would release its reference.

## Internal State & Concurrency
-   **State:** This class is **stateless and immutable**. Its only member, `MESSAGE_COMMANDS_DEBUG_SHAPE_SPHERE_SUCCESS`, is a static final constant. All operations within the `execute` method are performed on method-local variables or parameters, ensuring no state is carried between invocations.
-   **Thread Safety:** This class is thread-safe by design due to its stateless nature. However, it is **not safe** to invoke the `execute` method from an arbitrary thread. The command framework guarantees that it is called from the main server thread, which has safe access to the World and EntityStore. Direct, multi-threaded invocation would lead to severe race conditions and data corruption in the underlying game state systems. The use of `ThreadLocalRandom` further reinforces the expectation of single-threaded execution per command invocation.

## API Surface
The primary contract is the `execute` method, inherited and implemented from `AbstractPlayerCommand`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | protected void | O(1) | Executes the command logic. Fetches the player's position and instructs DebugUtils to render a sphere. Sends a success message via the CommandContext. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. It is invoked by the server's command system when a player types the corresponding command in the game console. The framework is responsible for constructing the `CommandContext` and providing all necessary dependencies.

The conceptual flow within the framework is as follows:
```java
// Conceptual example of framework invocation
// Do NOT replicate this code.

// 1. Command is parsed and matched to a registered instance
DebugShapeSphereCommand command = commandRegistry.find("sphere");

// 2. Context is built for the invoking player
CommandContext context = buildContextForPlayer(player);
Store<EntityStore> store = world.get(EntityStore.class);
Ref<EntityStore> ref = player.getEntityRef();
// ... and so on

// 3. The command is executed
command.execute(context, store, ref, player, world);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new DebugShapeSphereCommand()`. The command will not be registered with the server and will have no effect. All commands must be managed by the command registration system to function correctly.
-   **Manual Execution:** Do not call the `execute` method directly. Bypassing the command framework will skip critical steps like permission checks, context setup, and thread safety guarantees, likely leading to exceptions or server instability.
-   **Statefulness:** Do not add mutable instance fields to this class. Commands are expected to be stateless singletons; adding state can introduce bugs and concurrency issues across multiple command invocations.

## Data Pipeline
The flow of data for this command begins with player input and ends with a visible change in the game world.

> Flow:
> Player Input (`/sphere`) -> Server Command Parser -> Command Dispatcher -> **DebugShapeSphereCommand.execute()** -> ECS Query for `TransformComponent` -> `DebugUtils.addSphere()` -> World State Update -> Network Packet to Client for Rendering

