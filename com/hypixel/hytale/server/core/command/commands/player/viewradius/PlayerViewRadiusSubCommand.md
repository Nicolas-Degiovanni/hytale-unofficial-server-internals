---
description: Architectural reference for PlayerViewRadiusSubCommand
---

# PlayerViewRadiusSubCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.viewradius
**Type:** Transient

## Definition
```java
// Signature
public class PlayerViewRadiusSubCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The PlayerViewRadiusSubCommand class is a structural component within the server's Command System. It embodies the Composite design pattern, acting as a non-terminal node or a "branch" in the command hierarchy tree.

Its sole architectural purpose is to group related sub-commands—specifically PlayerViewRadiusGetCommand and PlayerViewRadiusSetCommand—under a single, logical parent command named `viewradius`. This class contains no executable logic itself. Instead, it serves as a routing mechanism. When the Command Dispatcher processes a command string like `/player viewradius set 12`, it first delegates to the `player` command, which in turn delegates to this `viewradius` collection. This class then identifies the next token, `set`, and passes execution control to the corresponding registered child command.

This pattern simplifies the command system by promoting a clean, hierarchical organization of related functionalities, preventing pollution of the top-level command namespace.

### Lifecycle & Ownership
-   **Creation:** Instantiated once during the server's bootstrap sequence. A higher-level command registry, likely the PlayerCommand, creates an instance of this class and registers it as one of its children. This process is part of the static assembly of the entire command tree.
-   **Scope:** Application-scoped. Once created and registered, the instance persists for the entire lifetime of the server process.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server shuts down and the root of the command system is dismantled.

## Internal State & Concurrency
-   **State:** Effectively immutable post-construction. The constructor populates its internal list of sub-commands, and this collection is not modified during runtime. Its state, consisting of its name and its children, is static.
-   **Thread Safety:** This class is inherently thread-safe. Because its internal state is fixed after initialization, it can be safely accessed by multiple command-processing threads concurrently without requiring any explicit locks or synchronization primitives. The parent AbstractCommandCollection is designed for safe, concurrent traversal.

## API Surface
The public contract of this class is almost entirely defined by its parent, AbstractCommandCollection. The only directly exposed public member is its constructor, which is used exclusively during system initialization. Interaction at runtime is handled through the abstract interface provided by the command system framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerViewRadiusSubCommand() | Constructor | O(1) | Initializes the command collection with a fixed name and registers its child commands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by feature developers. It is a framework component registered declaratively. The standard pattern is to add it as a sub-command to a parent collection.

```java
// Example from a hypothetical parent command's constructor
// This is how PlayerViewRadiusSubCommand is integrated into the system.
public class PlayerCommand extends AbstractCommandCollection {
    public PlayerCommand() {
        super("player", "...");
        
        // Registering the viewradius command group
        this.addSubCommand(new PlayerViewRadiusSubCommand());
        // ... other player-related sub-commands
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Runtime Modification:** Do not attempt to retrieve this object at runtime to add or remove sub-commands. The command tree is considered static after server initialization. Modifying it can lead to race conditions and undefined behavior in command dispatch.
-   **Direct Instantiation:** Never instantiate this class with `new` outside of the initial command registration phase. The command system relies on a single, statically-defined tree.

## Data Pipeline
As a routing component, PlayerViewRadiusSubCommand does not process data. Instead, it directs the flow of command execution.

> Flow:
> Player Chat Input -> Command Parser -> Command Dispatcher -> `PlayerCommand` -> **`PlayerViewRadiusSubCommand`** -> `PlayerViewRadiusGet/SetCommand` -> Command Executor

