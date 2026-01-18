---
description: Architectural reference for PlayerCommand
---

# PlayerCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player
**Type:** Transient

## Definition
```java
// Signature
public class PlayerCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The PlayerCommand class is a foundational component of the server's command system, acting as a **Command Collection** rather than a command that executes logic itself. Its primary architectural role is to group all player-related administrative sub-commands under a single, user-facing namespace: `/player`.

This class implements the **Composite design pattern**. It serves as a composite node in the command tree, holding a collection of leaf nodes (the individual sub-commands like PlayerResetCommand, PlayerStatsSubCommand, etc.). When the server's command dispatcher receives a command beginning with `/player`, it delegates processing to this object. PlayerCommand then parses the subsequent arguments to identify and execute the appropriate sub-command from its internal registry.

This design decouples the root command from its children, allowing for modular and extensible architecture. New player-related functionality can be added by creating a new sub-command class and registering it within the PlayerCommand constructor, without modifying the core command parsing or dispatching logic.

### Lifecycle & Ownership
- **Creation:** A single instance of PlayerCommand is created by the server's central command registration service during the server bootstrap sequence. This is an automated process, typically driven by class-path scanning for command definitions.
- **Scope:** The object instance is stateless beyond its initial configuration. It is registered with and owned by the server's global CommandRegistry, where it persists for the entire server session.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server shuts down and the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** The internal state consists of a list of registered sub-command objects. This state is populated exclusively within the constructor. Once the object is instantiated, its collection of sub-commands is considered effectively immutable. This is a "write-once, read-many" pattern.
- **Thread Safety:** The class is inherently thread-safe for execution. The collection of sub-commands is populated on the main server thread at startup, eliminating any risk of modification race conditions during runtime. The underlying `AbstractCommandCollection` guarantees that concurrent read access to the sub-command list during command execution is safe.

## API Surface
The public API of PlayerCommand is entirely inherited from its parent, `AbstractCommandCollection`. Direct interaction with a PlayerCommand instance is not a standard operational pattern; interaction is mediated through the server's command processing system. The constructor serves as the configuration point for developers extending the system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerCommand() | constructor | O(N) | Instantiates the collection and registers N default sub-commands. Intended for framework use only. |

## Integration Patterns

### Standard Usage
The primary integration pattern is extension. Developers should not interact with this class directly but rather add new functionality to it. To create a new `/player` sub-command, a developer would create a new command class and add it to the collection within the constructor.

```java
// Example: Adding a new hypothetical "heal" sub-command
public class PlayerCommand extends AbstractCommandCollection {
   public PlayerCommand() {
      super("player", "server.commands.player.desc");
      this.addSubCommand(new PlayerResetCommand());
      this.addSubCommand(new PlayerStatsSubCommand());
      // ... existing commands
      
      // Add the new command here
      this.addSubCommand(new PlayerHealSubCommand());
   }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new PlayerCommand()` in game logic. The server framework is responsible for the creation and registration of all command objects. Manual instantiation will result in a disconnected command that is not registered with the server.
- **Runtime Modification:** Do not attempt to retrieve the PlayerCommand instance from the command registry at runtime to add or remove sub-commands. The command collection is not designed for dynamic modification and doing so can lead to severe concurrency issues and unpredictable behavior.

## Data Pipeline
PlayerCommand acts as a dispatcher in the server's command processing pipeline. It receives control after the initial command token has been identified and is responsible for delegating control to the correct sub-system.

> Flow:
> Client Input (`/player stats HytaleFan`) -> Server Network Handler -> Command Parser -> CommandRegistry (resolves "player") -> **PlayerCommand** (resolves "stats") -> PlayerStatsSubCommand (executes logic) -> Game State Mutation -> Network Response Packet -> Client UI Update

