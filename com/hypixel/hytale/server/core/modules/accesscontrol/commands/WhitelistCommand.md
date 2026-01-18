---
description: Architectural reference for WhitelistCommand
---

# WhitelistCommand

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.commands
**Type:** Transient

## Definition
```java
// Signature
public class WhitelistCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The WhitelistCommand class is a **Command Aggregator**. It does not implement any direct command logic itself. Instead, its sole responsibility is to act as a container and entry point for a suite of related sub-commands that manage the server's whitelist functionality.

Architecturally, it follows the **Composite Pattern**. It is a concrete implementation of AbstractCommandCollection, grouping leaf-node commands (like WhitelistAddCommand, WhitelistRemoveCommand) under a single, user-facing command: *whitelist*.

When a player or administrator executes a command like `/whitelist add PlayerName`, the server's central Command Dispatcher routes the request to this WhitelistCommand instance. The class then delegates the execution to the appropriate registered sub-command based on the first argument ("add").

This design decouples the parent command from the specific implementation of its children, promoting modularity. All business logic is further delegated from the sub-commands to the injected **HytaleWhitelistProvider**, ensuring a clean separation of concerns between the command interface and the access control service.

### Lifecycle & Ownership
- **Creation:** An instance of WhitelistCommand is created during server startup, typically within the initialization phase of an access control module. It requires a valid HytaleWhitelistProvider instance to be injected via its constructor.
- **Scope:** The object's lifecycle is tied to the server session. It is created once when the server starts and persists until the server shuts down.
- **Destruction:** The object is eligible for garbage collection upon server shutdown or when the command registry that holds a reference to it is cleared.

## Internal State & Concurrency
- **State:** WhitelistCommand is effectively **stateless and immutable** post-construction. Its internal state consists of a list of its sub-commands, which is populated exclusively within the constructor and is not modified thereafter. The actual whitelist data is managed by the HytaleWhitelistProvider service, not this class.
- **Thread Safety:** This class is inherently thread-safe. As it holds no mutable state, it can be safely accessed from any thread. However, command execution is typically marshaled onto the main server thread to prevent concurrency issues with game state. The thread safety of the underlying whitelist system is the responsibility of the HytaleWhitelistProvider.

## API Surface
The primary programmatic interaction with this class is its constructor for dependency injection. The public API is otherwise defined by its parent, AbstractCommandCollection, for use by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WhitelistCommand(provider) | constructor | O(1) | Constructs the command group. Injects the HytaleWhitelistProvider dependency into all registered sub-commands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation. It is designed to be instantiated and registered with the server's central command management system during the bootstrap process.

```java
// Example from a module's initialization method
HytaleWhitelistProvider provider = context.getService(HytaleWhitelistProvider.class);
CommandManager commandManager = context.getService(CommandManager.class);

// Create the command group and register it
commandManager.register(new WhitelistCommand(provider));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation without Registration:** Creating an instance via `new WhitelistCommand(...)` without subsequently registering it with the server's CommandManager will result in a "dangling" object that is never used. The `/whitelist` command will not be available.
- **Null Provider Injection:** The constructor is annotated with Nonnull. Attempting to instantiate this class with a null HytaleWhitelistProvider will violate its contract and lead to a NullPointerException, causing a server fault during startup.

## Data Pipeline
The WhitelistCommand acts as a routing step in the server's command processing pipeline. It receives a parsed command and dispatches it to the correct sub-component.

> Flow:
> Player Input (`/whitelist status`) -> Network Packet -> Command Parser -> Command Dispatcher -> **WhitelistCommand** -> WhitelistStatusCommand -> HytaleWhitelistProvider -> Command Result -> Network Packet -> Player Chat Response

