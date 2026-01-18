---
description: Architectural reference for WorldPerfGraphCommand
---

# WorldPerfGraphCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.world.perf
**Type:** Transient

## Definition
```java
// Signature
public class WorldPerfGraphCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WorldPerfGraphCommand is a concrete implementation of the Command Pattern, designed to function as a diagnostic tool within the server's command processing system. It is a specialized leaf node in the `/world perf` command hierarchy.

Architecturally, this class serves as a bridge between the server's low-level performance monitoring subsystem and a human operator. It queries the active World instance for its HistoricMetric data, which contains buffered, time-series information about server tick durations. The command's primary responsibility is to transform this raw numerical data into a human-readable, text-based graph, providing an intuitive visualization of server performance over various time windows.

This command is not a long-lived service or manager. It is a stateless handler, instantiated by the command framework and invoked to process a single, specific user request. Its existence decouples the logic for performance data visualization from the core World simulation and the command parsing engine.

## Lifecycle & Ownership
- **Creation:** A single instance of WorldPerfGraphCommand is instantiated by the server's central command registration system during the server bootstrap sequence. It is then registered under the path `world perf graph`.
- **Scope:** The object instance persists for the entire server session. It is held as a reference within the command registry's internal data structures.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the server shuts down or the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless. The fields `widthArg` and `heightArg` are final and define the command's argument structure, not its runtime state. They are configured once in the constructor and are immutable thereafter. The `execute` method operates exclusively on local variables and parameters passed into it, ensuring that each command execution is independent and isolated.

- **Thread Safety:** This command is not thread-safe and is not designed to be. The server's command system is responsible for ensuring that all command execution occurs on the main server thread. The `execute` method directly accesses the `World` object to retrieve performance metrics. Calling `world.getBufferedTickLengthMetricSet()` from an external thread would violate the server's threading model and lead to severe concurrency issues. The use of a *buffered* metric set is a deliberate design choice to provide a safe, point-in-time snapshot of performance data to the main thread without requiring locks that would stall the metrics collector.

## API Surface
The public contract is fulfilled by overriding the `execute` method from its parent, AbstractWorldCommand.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(W) | Entry point for the command framework. Orchestrates the fetching, processing, and rendering of performance data. Complexity is linear relative to the requested graph width (W). |

## Integration Patterns

### Standard Usage
Developers do not typically invoke this class directly. Instead, an instance is registered with the command system, which handles its lifecycle and invocation.

```java
// Conceptual example of command registration during server startup
CommandSystem commandSystem = server.getCommandSystem();
commandSystem.register(new WorldPerfGraphCommand());

// The command is then invoked by a user via the console or chat:
// > /world perf graph width=120 height=15
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Execution:** Never create and call this command manually. Doing so bypasses the entire command framework, including argument parsing, permission checks, and context provision. The `CommandContext` object is critical for execution and can only be supplied correctly by the framework.
- **Stateful Modification:** Do not modify this class to hold state in its instance fields. The same object instance is used for every execution of the command. Introducing mutable instance state would create race conditions and unpredictable behavior between subsequent command calls.

## Data Pipeline
This command processes data by transforming low-level server metrics into a user-facing text visualization.

> Flow:
> User Input (`/world perf graph...`) -> Command Framework Parser -> **WorldPerfGraphCommand.execute** -> World.getBufferedTickLengthMetricSet() -> StringUtil.generateGraph -> Message Builder -> CommandContext.sendMessage -> Formatted Chat Output

