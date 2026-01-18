---
description: Architectural reference for VoidEventCommands
---

# VoidEventCommands

**Package:** com.hypixel.hytale.builtin.portals.commands.voidevent
**Type:** Configuration Object

## Definition
```java
// Signature
public class VoidEventCommands extends AbstractCommandCollection {
```

## Architecture & Concepts
The VoidEventCommands class is not a command itself, but rather a structural component that functions as a container for a group of related server commands. By extending AbstractCommandCollection, it participates in the server's command system as a top-level namespace, in this case, **voidevent**.

Its primary architectural role is to enforce modularity and organization. Instead of registering every individual command with the global command registry, developers create collections that bundle commands by feature. The server's CommandManager introspects these collections during startup to build a hierarchical command tree. This pattern decouples the command implementation (e.g., StartVoidEventCommand) from the global registration process, simplifying the addition and removal of command sets.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's command loading mechanism during the bootstrap or feature initialization phase. The system scans for classes of this type and creates a single instance to be registered.
- **Scope:** The object instance persists for the entire server session. It is held by the central CommandRegistry to maintain the definition of the command tree.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the server shuts down and the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** The state of this object is effectively **immutable** after construction. The list of subcommands is populated exclusively within the constructor and is not designed for modification at runtime. The parent class, AbstractCommandCollection, manages this internal list.
- **Thread Safety:** This class is **conditionally thread-safe**. It is safe for multiple threads to interact with the commands it defines *after* the server's initialization phase is complete. The construction and registration process itself is not thread-safe and must be performed in a single-threaded context during server startup.

## API Surface
The public contract of this class is limited to its constructor, which is invoked by the engine. It has no other meaningful public API intended for developers to call directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| VoidEventCommands() | Constructor | O(N) | Constructs the command collection, setting its root name and registering N subcommands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by game logic or plugin developers. It is a declarative definition that is discovered and consumed by the server's core command system. A developer would typically create a similar class to define their own command suite.

A hypothetical registration process managed by the engine would look like this:
```java
// This logic resides within the server's CommandManager or equivalent service.
// It is not intended to be called by external code.
CommandRegistry registry = server.getCommandRegistry();
registry.register(new VoidEventCommands());
```

### Anti-Patterns (Do NOT do this)
- **Runtime Instantiation:** Do not create an instance of VoidEventCommands after the server has started. The command tree is considered finalized post-initialization.
- **State Mutation:** Do not attempt to retrieve a registered collection and modify its subcommands at runtime. This can lead to unpredictable behavior and race conditions within the command dispatcher.

## Data Pipeline
This class acts as a configuration point that defines a branch in the command processing pipeline. It does not process data itself but directs the flow of command data to the appropriate handler.

> Flow:
> Player Chat Input (`/voidevent start`) -> Network Layer -> Command Parser -> **Command Tree (branch defined by VoidEventCommands)** -> StartVoidEventCommand Executor

