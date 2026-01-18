---
description: Architectural reference for AbstractCommandCollection
---

# AbstractCommandCollection

**Package:** com.hypixel.hytale.server.core.command.system.basecommands
**Type:** Framework Component

## Definition
```java
// Signature
public abstract class AbstractCommandCollection extends AbstractAsyncCommand {
```

## Architecture & Concepts
The AbstractCommandCollection is a foundational component within the server's command processing framework. Its primary architectural purpose is to serve as a non-executable container or namespace for a group of related sub-commands. It enables the creation of hierarchical command structures, such as `/permission group <group> set <node>`. In this example, `/permission` would be a concrete implementation of AbstractCommandCollection.

Unlike standard commands that perform a specific action, this class intentionally overrides the core execution logic to provide help and usage information. When a player or system executes a command that is an AbstractCommandCollection, the framework does not perform an action. Instead, it automatically displays a list of available sub-commands and their usage, guiding the user on how to properly interact with the command family.

By extending AbstractAsyncCommand, it integrates seamlessly into the asynchronous command execution pipeline, even though its own operation is synchronous and completes instantaneously. This design enforces a consistent and user-friendly command interface, preventing ambiguity and improving discoverability of server features.

### Lifecycle & Ownership
- **Creation:** Concrete subclasses of AbstractCommandCollection are instantiated once during server initialization. This is typically handled by a dependency injection framework or a dedicated CommandRegistry service that scans for and registers all command definitions.
- **Scope:** A command collection instance is a long-lived object. Once registered, it persists for the entire server runtime, acting as a stateless service.
- **Destruction:** The object is dereferenced and becomes eligible for garbage collection only during server shutdown when the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. Its internal fields for name and description are initialized via the constructor and are not modified during its lifecycle. It does not cache any request-specific data.
- **Thread Safety:** The class is inherently **thread-safe**. Due to its stateless nature, a single instance can be safely invoked by multiple threads concurrently without risk of race conditions or data corruption. All operations are performed on the `CommandContext` object, which is scoped to a single, isolated invocation. No internal locking mechanisms are necessary.

## API Surface
The public contract is minimal, focusing on providing usage information. The core execution logic is `protected final` and not part of the public API for implementers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFullUsage(sender) | Message | O(N) | Returns a comprehensive usage message, typically listing all registered sub-commands. N is the number of sub-commands. |
| getUsageString(sender) | Message | O(N) | Returns a more concise usage message. Behavior is inherited and may be customized by subclasses. |

**WARNING:** The `executeAsync` method is `final`. Subclasses cannot and must not attempt to override it. The framework relies on this behavior to ensure that collection-type commands consistently provide help text instead of performing actions.

## Integration Patterns

### Standard Usage
Developers should extend this class to create a new command namespace. Sub-commands are then registered to this collection through the command system's registration API.

```java
// 1. Define the collection command
public class GameModeCommand extends AbstractCommandCollection {
    public GameModeCommand() {
        super("gamemode", "Sets a player's game mode.");
        // Sub-commands like 'creative', 'survival' would be
        // programmatically associated with this collection
        // by the CommandRegistry.
    }
}

// 2. The framework handles invocation
// When a player types "/gamemode", the final executeAsync method
// in AbstractCommandCollection is called, which prints usage.
```

### Anti-Patterns (Do NOT do this)
- **Adding Execution Logic:** Do not add action-performing logic to a subclass of AbstractCommandCollection. Its sole purpose is to group other commands. If you need a command that has a default action but also has sub-commands, a different base class should be used.
- **Stateful Implementations:** Avoid adding mutable fields to subclasses. Commands must remain stateless to ensure thread safety and predictable behavior across the server.
- **Direct Instantiation:** Never instantiate a command using `new GameModeCommand()`. The server's CommandRegistry is responsible for the lifecycle of all command objects.

## Data Pipeline
The AbstractCommandCollection acts as a terminal node in the command dispatch pipeline when it is the final token in a command string. It intercepts the execution flow to return help information instead of performing a game world action.

> Flow:
> Player Input (`/gamemode`) -> Network Layer -> Command Parser -> Command Dispatcher -> **AbstractCommandCollection::executeAsync** -> Message Builder -> CommandSender -> Player's Chat UI

