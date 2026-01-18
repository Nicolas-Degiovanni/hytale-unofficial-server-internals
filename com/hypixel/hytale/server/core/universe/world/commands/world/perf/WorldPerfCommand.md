---
description: Architectural reference for WorldPerfCommand
---

# WorldPerfCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.world.perf
**Type:** Handler

## Definition
```java
// Signature
public class WorldPerfCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WorldPerfCommand class is a server-side administrative command handler responsible for exposing real-time world performance metrics. It functions as a diagnostic tool for server operators, providing insight into tick rate (TPS) and tick processing time.

Architecturally, this class is a node within the server's hierarchical command system. It is registered as a subcommand of a primary `world` command. Its core responsibility is to act as a bridge between a user-initiated request and the engine's internal metric collection service. It queries the target World for its `HistoricMetric` data, formats this raw data into a human-readable report, and sends it back to the command's sender.

This command is also a composite, acting as a parent for more specialized performance commands such as WorldPerfGraphCommand and WorldPerfResetCommand, delegating execution to them when their respective sub-paths are invoked.

## Lifecycle & Ownership
- **Creation:** A single instance of WorldPerfCommand is created during server initialization by the Command System. The system scans for command implementations and instantiates them, registering them for later invocation.
- **Scope:** Application-scoped. The singleton instance persists for the entire lifetime of the server process. It does not hold per-request state.
- **Destruction:** The instance is de-referenced and eligible for garbage collection only upon server shutdown when the central command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its fields, `allFlag` and `deltaFlag`, are final and initialized during construction. All data required for execution is passed into the `execute` method via the CommandContext and World parameters. It does not cache data or maintain state across invocations.
- **Thread Safety:** The `execute` method is designed to be called by the server's main game thread, which has exclusive write access to the World object. The method reads performance data from a `HistoricMetric` object, which is explicitly designed for safe, buffered reads from a high-contention source. Therefore, the command handler itself is thread-safe under the engine's expected threading model. The static utility methods are pure functions and are inherently thread-safe.

## API Surface
The primary contract is fulfilled by overriding the `execute` method from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | protected void | O(1) | The command's entry point. Gathers, formats, and reports world performance metrics. Complexity is constant as it iterates over a fixed number of time periods. |
| tpsFromDelta(delta, min) | public static double | O(1) | A stateless utility function to calculate Ticks Per Second from a nanosecond delta time. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is invoked automatically by the server's command processing system in response to user input.

A server operator or a player with sufficient permissions would execute the command via the chat console.

```
# User input in the game client or server console
/world perf --delta
```

The system then routes this to the handler:
```java
// Conceptual representation of how the Command System invokes this handler.
// Do not replicate this pattern; it is managed by the engine.
CommandContext ctx = commandSystem.createContextFor("/world perf --delta");
WorldPerfCommand handler = commandSystem.getHandlerFor(ctx);
World targetWorld = ctx.getWorld();
Store<EntityStore> store = targetWorld.getEntityStore();

handler.execute(ctx, targetWorld, store);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WorldPerfCommand()`. The command system manages the lifecycle of all command handlers. Manual instantiation will result in a non-functional object that is not registered to receive commands.
- **Manual Invocation:** Avoid calling the `execute` method directly. Doing so bypasses the engine's critical infrastructure for argument parsing, permission checking, and context resolution, leading to unpredictable behavior and potential exceptions.

## Data Pipeline
The flow of data for a typical invocation is unidirectional, from the engine's core metrics to the user's screen.

> Flow:
> User Input (`/world perf`) -> Command Parser -> **WorldPerfCommand.execute()** -> World.getBufferedTickLengthMetricSet() -> Message Builder -> CommandContext.sendMessage() -> Server Network Layer -> Client Chat UI

