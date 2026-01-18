---
description: Architectural reference for BlockCommand
---

# BlockCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.block
**Type:** Transient

## Definition
```java
// Signature
public class BlockCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The BlockCommand class is a structural component within the server's command handling system. It does not implement any executable logic itself; instead, it functions as a **Composite Command Group**. Its primary architectural role is to act as a namespace and entry point for a collection of related sub-commands that perform actions on world blocks.

By inheriting from AbstractCommandCollection, it integrates into the command dispatch tree. When a player executes a command beginning with `/block` or `/blocks`, the server's CommandDispatcher routes the request to this object. BlockCommand then parses the next argument (e.g., "set", "get", "bulk") and delegates the remainder of the command string to the corresponding registered sub-command for final execution.

This pattern simplifies command registration, promotes code organization by grouping related functionalities, and provides a consistent user-facing command structure.

## Lifecycle & Ownership
- **Creation:** A single instance of BlockCommand is created during the server's bootstrap sequence. This is typically handled by a central `CommandRegistry` service that scans for and instantiates all command definitions.
- **Scope:** The object is retained by the CommandRegistry for the entire duration of the server session. It is a long-lived but passive object.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server shuts down and the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** The state of BlockCommand is configured once within its constructor and is **effectively immutable** thereafter. This includes its name, aliases, required permission level, and the static list of its sub-commands. It holds no mutable runtime state and performs no caching.
- **Thread Safety:** This class is inherently **thread-safe**. Because its internal state is fixed upon construction, it can be safely accessed from multiple threads without synchronization. Any concurrency concerns related to command execution are the responsibility of the downstream sub-commands (e.g., BlockSetCommand) and the world-modification systems they interact with.

## API Surface
The public contract of this class is fulfilled almost entirely by its constructor. It has no significant public methods intended for direct invocation after instantiation. Its behavior is defined by its configuration and its integration with the parent AbstractCommandCollection class, which is used internally by the command dispatch system.

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is automatically discovered and registered by the server's command system at startup. The standard interaction is performed by a server administrator or a player in Creative Mode through the in-game console.

**User Interaction Example:**
```
/block set 100 64 100 hytale:stone
```
In this example, the command system routes to the BlockCommand instance, which in turn delegates to the registered BlockSetCommand to perform the world modification.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create instances of this class manually using `new BlockCommand()`. An unregistered instance is inert and will not be recognized by the server's command dispatcher. All command registration must occur through the designated bootstrap mechanism.
- **Dynamic Modification:** Do not attempt to add or remove sub-commands from this collection after the server has started. The command tree is built once and is not designed for runtime mutation. Doing so will lead to unpredictable behavior and potential server instability.

## Data Pipeline
BlockCommand acts as a routing node in the data pipeline for player-issued commands. It validates the primary command token and directs the flow to the appropriate handler.

> Flow:
> Player Input String (`/block ...`) -> Server Network Listener -> CommandDispatcher -> **BlockCommand** (Routing & Permission Check) -> Sub-Command (e.g., BlockSetCommand) -> World API -> Chunk Modification

