---
description: Architectural reference for StressTestCommand
---

# StressTestCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.stresstest
**Type:** Transient

## Definition
```java
// Signature
public class StressTestCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The StressTestCommand class serves as a structural component within the server's command system. It is not an executable command itself but rather a **Command Group** that acts as a namespace for a collection of related sub-commands. Its primary architectural role is to organize and route command invocations.

This class implements the **Composite** design pattern. It functions as a non-terminal node in the command tree, holding references to leaf nodes (StressTestStartCommand, StressTestStopCommand). When the server's command parser identifies the "stresstest" token, it delegates responsibility to this object, which in turn dispatches the request to the appropriate sub-command based on subsequent input tokens.

By aggregating functionality, this class simplifies the command landscape for both users and developers, grouping all stress-testing operations under a single, predictable entry point.

### Lifecycle & Ownership
- **Creation:** An instance of StressTestCommand is created during the server's bootstrap sequence. A central `CommandRegistry` service is responsible for discovering and instantiating all command classes, including this one.
- **Scope:** The object instance is held within the central `CommandRegistry` for the entire duration of the server session. It is effectively a singleton within the scope of the command system.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the `CommandRegistry` is cleared.

## Internal State & Concurrency
- **State:** The state of this class is considered **immutable after construction**. The list of sub-commands is populated exclusively within the constructor and is not modified at any point during its lifecycle. All state is managed by the parent AbstractCommandCollection.
- **Thread Safety:** This class is inherently thread-safe for read operations. Since its internal state is fixed post-construction, multiple threads can safely traverse its sub-command list during command parsing. However, command *execution* is a separate concern, typically managed and serialized by the core command processing system to prevent race conditions in downstream logic.

## API Surface
The public contract of this class is minimal, as its primary interaction is with the command registration system via its constructor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| StressTestCommand() | Constructor | O(1) | Constructs the command group, defining its name and registering its child sub-commands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct method invocation. Its standard use is to be instantiated and passed to the server's command registry during initialization.

```java
// Example from a hypothetical Command Registration Service
CommandRegistry commandRegistry = server.getCommandRegistry();

// The StressTestCommand is instantiated and immediately registered.
// The registry now owns the instance.
commandRegistry.register(new StressTestCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create an instance of StressTestCommand for any purpose other than immediate registration with the `CommandRegistry`. The object has no utility on its own and will be garbage collected if not held by the registry.
- **Adding Execution Logic:** Avoid overriding methods to add direct execution logic to this class. Its sole responsibility is to act as a container. All business logic must be encapsulated within distinct sub-command classes. Modifying this class to execute logic violates the Single Responsibility Principle.

## Data Pipeline
StressTestCommand acts as a routing step in the server's command processing pipeline. It receives control from the main parser and delegates it to a more specific handler.

> Flow:
> User Input (`/stresstest start`) -> Command Parser -> **StressTestCommand** (Router) -> StressTestStartCommand (Executor) -> Server Stress Test Service

