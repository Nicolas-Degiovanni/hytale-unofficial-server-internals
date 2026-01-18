---
description: Architectural reference for FragmentCommands
---

# FragmentCommands

**Package:** com.hypixel.hytale.builtin.portals.commands
**Type:** Transient

## Definition
```java
// Signature
public class FragmentCommands extends AbstractCommandCollection {
```

## Architecture & Concepts
The FragmentCommands class serves as a structural container within the server's command processing system. It embodies the **Composite Command** pattern, grouping a set of related subcommands under a single, top-level command namespace, in this case, *fragment*.

This class does not implement any command logic itself. Its sole responsibility is to declare a primary command name ("fragment") and aggregate one or more subcommand implementations, such as TimerFragmentCommand. The server's central Command Dispatcher uses this collection to route incoming command strings. For example, a player input of `/fragment timer ...` would first be routed to this collection based on the "fragment" token, which then delegates processing to the appropriate subcommand handler.

It is a foundational piece of the command system's extensibility, allowing developers to logically group and register new families of commands without modifying the core command dispatcher.

## Lifecycle & Ownership
- **Creation:** Instantiated once by the server's core command registration service during the bootstrap sequence. This is not an object that should ever be created by game logic.
- **Scope:** The instance is registered and held by the central command registry. It persists for the entire duration of the server session.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** The state of this object is effectively **immutable** after its constructor completes. The internal list of subcommands is populated during instantiation and is not designed to be modified at runtime.
- **Thread Safety:** This class is inherently **thread-safe**. Its state is established on a single thread during server initialization and is not mutated thereafter. The execution of the contained commands is managed by the broader command system, which is responsible for handling concurrent command execution from multiple players. This class itself is not involved in the execution lifecycle and poses no concurrency risks.

## API Surface
The public contract of this class is limited to its constructor, which is used exclusively for initialization and registration by the framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FragmentCommands() | Constructor | O(N) | Initializes the command collection, setting its primary name and registering all N subcommands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use. It is discovered and instantiated by the command system. A developer seeking to add a new command family would follow this pattern:

```java
// In a central registration service during server startup:
CommandRegistry registry = server.getCommandRegistry();

// The system instantiates and registers the collection.
// This is the only valid interaction pattern.
registry.register(new FragmentCommands());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never instantiate this class within game logic using `new FragmentCommands()`. Instances created this way will not be registered with the command system and will have no effect.
- **State Mutation:** Do not attempt to use reflection or other means to modify the internal subcommand list after initialization. The command system does not expect or support dynamic changes to command collections at runtime, which can lead to server instability.

## Data Pipeline
FragmentCommands acts as a routing node in the server's command processing pipeline. It does not transform data but directs it to the correct handler.

> Flow:
> Player Chat Input -> Network Packet -> Command Parser -> Command Dispatcher -> **FragmentCommands** (Route by "fragment") -> TimerFragmentCommand (Execute) -> Game World State Change

