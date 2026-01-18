---
description: Architectural reference for PrefabPathCommand
---

# PrefabPathCommand

**Package:** com.hypixel.hytale.builtin.path.commands
**Type:** Component

## Definition
```java
// Signature
public class PrefabPathCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The PrefabPathCommand class is a **Command Aggregator**. It does not implement any command logic itself. Instead, its sole architectural purpose is to act as a container and entry point for a suite of related sub-commands that manage server-side NPC pathing prefabs.

This class implements the **Composite** design pattern, where it serves as a composite node in the server's command tree. It registers itself with the server's command system under the primary alias "path". When a user executes a command like `/path list`, the command system first routes the request to this PrefabPathCommand instance. The instance then delegates the final execution to the appropriate child command (in this case, PrefabPathListCommand) based on the subsequent arguments.

This pattern provides a clean, hierarchical namespace for server commands, improving discoverability and maintainability by grouping related functionalities.

### Lifecycle & Ownership
- **Creation:** An instance of PrefabPathCommand is created by the server's command registration system during the server bootstrap sequence. The system typically scans the classpath for command implementations and instantiates them.
- **Scope:** The object's lifecycle is tied to the server session. It is instantiated once upon server start and persists in the central command registry until the server shuts down.
- **Destruction:** The object is eligible for garbage collection only upon server shutdown when the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively **stateless and immutable** after construction. Its internal state consists of a list of sub-command objects. This list is populated exclusively within the constructor and is not modified thereafter.
- **Thread Safety:** The class is inherently thread-safe. Command execution in the Hytale server is managed by a dedicated system that processes commands sequentially, preventing concurrent modification or access issues. As this object's internal collection is write-protected after initialization, no external locking is necessary.

## API Surface
The public contract of this class is fulfilled by its constructor. All other behavior is inherited from the AbstractCommandCollection base class and is handled by the command system framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PrefabPathCommand() | Constructor | O(k) | Initializes the command group. Sets the primary alias to "path" and populates the internal collection with all path-related sub-commands. *k* is the number of sub-commands. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. It is designed to be automatically discovered and registered by the server's command system. The primary interaction is from a server administrator or a user with appropriate permissions via the in-game console.

A user would execute commands like:
- `/path list`
- `/path new my_path_name`
- `/path edit my_path_name`

The system-level registration would look conceptually similar to this:
```java
// Conceptual example of server-side registration
CommandSystem commandSystem = server.getCommandSystem();
commandSystem.register(new PrefabPathCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create an instance of PrefabPathCommand for any purpose other than registration with the core CommandSystem. Instantiating it without registration has no effect.
- **Runtime Modification:** Do not attempt to retrieve this object from the command registry to add or remove sub-commands at runtime. The command hierarchy is intended to be static for the duration of a server session.

## Data Pipeline
This class acts as a dispatcher in the server's command processing pipeline. It receives a partially parsed command and routes it to the correct handler.

> Flow:
> User Input (`/path list`) -> Server Network Listener -> Command Parser -> Command Dispatcher -> **PrefabPathCommand** -> PrefabPathListCommand -> World State -> Command Result -> Network Serializer -> Client Console

