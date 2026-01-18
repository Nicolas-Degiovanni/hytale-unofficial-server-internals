---
description: Architectural reference for LightingInfoCommand
---

# LightingInfoCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.lighting
**Type:** Transient

## Definition
```java
// Signature
public class LightingInfoCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The LightingInfoCommand is a concrete implementation of the Command Pattern, designed to function as a server-side diagnostic tool. It integrates with the server's command processing framework to provide administrators and developers with a read-only snapshot of the world's lighting engine state.

By extending AbstractWorldCommand, this class delegates the responsibility of context validation to the framework. The framework ensures that the command is only executed when a valid World instance is available, preventing null-pointer exceptions and ensuring a consistent execution environment.

Architecturally, this command acts as a pure data inspector. It does not modify world state or trigger any calculations. Its sole purpose is to query authoritative systems—primarily the ChunkLightingManager and the ChunkStore—and report their current status. This maintains a clean separation of concerns, where diagnostic tools are decoupled from the systems they observe.

## Lifecycle & Ownership
- **Creation:** A single prototype instance of LightingInfoCommand is instantiated by the server's command registration system during the bootstrap sequence. This instance is stored within a central command registry.

- **Scope:** The prototype instance persists for the entire server session, managed by the command registry. However, the *execution* of the command is ephemeral. Each invocation of the command via the server console or chat initiates a short-lived operation that terminates upon sending the response message.

- **Destruction:** The prototype instance is marked for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless from an execution perspective. Its only member field, detailFlag, is an immutable configuration object initialized in the constructor. All data required for execution is passed as arguments to the execute method.

- **Thread Safety:** The class is thread-safe. The detailed report generation leverages a parallel stream (forEachEntityParallel) to iterate over world chunks. To prevent race conditions during aggregation, it correctly employs AtomicInteger for its counters. This design allows for efficient, non-blocking inspection of large worlds without compromising data integrity.

    **Warning:** The underlying systems, such as ChunkStore, are assumed to be thread-safe for read operations as invoked by this command.

## API Surface
The public contract is defined by the inherited abstract method from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) or O(N) | Entry point for command execution. Complexity is O(1) for the summary report. If the detail flag is active, complexity becomes O(N), where N is the total number of BlockSections in the world. |

## Integration Patterns

### Standard Usage
This command is intended to be invoked by a server administrator through the server console or in-game chat. The framework supplies the necessary context.

```java
// This code is conceptual. A user would type the command into a console.
// Framework-level invocation:
CommandContext ctx = ...; // Provided by the server
World world = ...; // Provided by the server
Store<EntityStore> store = ...; // Provided by the server

LightingInfoCommand cmd = commandRegistry.get("lighting info");
cmd.execute(ctx, world, store);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually create instances using `new LightingInfoCommand()`. The command is useless without the context provided by the command execution framework. It cannot be executed as a standalone object.

- **Stateful Execution:** Do not attempt to store state in member variables across multiple `execute` calls. The framework makes no guarantees about which thread will execute the command or if the same instance is used for every invocation.

- **Unsafe Parallelism:** When modifying this command, avoid replacing AtomicInteger with primitive types like `int` in the parallel stream. Doing so will introduce race conditions and lead to inaccurate reports.

## Data Pipeline
LightingInfoCommand is a terminal component in a data inspection pipeline. It initiates data reads but does not propagate data to other systems.

> Flow:
> User Input (`/lighting info --detail`) -> Server Command Parser -> **LightingInfoCommand** -> Read `ChunkLightingManager` State -> Read `ChunkStore` Data -> Aggregate Metrics -> Format `Message` -> Send to `CommandContext` -> User Console Output

