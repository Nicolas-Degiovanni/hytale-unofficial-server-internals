---
description: Architectural reference for ParticleCommand
---

# ParticleCommand

**Package:** com.hypixel.hytale.server.core.asset.type.particle.commands
**Type:** Transient

## Definition
```java
// Signature
public class ParticleCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The ParticleCommand class serves as a structural grouping mechanism within the server's command dispatch system. It does not implement any executable logic itself; instead, it acts as a parent node or namespace for a collection of related subcommands, all prefixed by the name *particle*.

By extending AbstractCommandCollection, it inherits the responsibility of routing command execution to the appropriate subcommand. When the server's central CommandManager processes a command string like `/particle spawn ...`, it first identifies this ParticleCommand object as the handler for the `particle` token. This class then takes responsibility for parsing the remainder of the command string and delegating execution to the registered ParticleSpawnCommand instance.

This pattern is critical for organizing the command system, preventing namespace collisions, and creating a hierarchical, user-friendly command structure.

### Lifecycle & Ownership
- **Creation:** Instantiated once during the server's bootstrap sequence. A central CommandManager or a similar registry service is responsible for discovering and creating an instance of this class to register it in the server's command map.
- **Scope:** Session-scoped. The single instance persists for the entire lifetime of the running server.
- **Destruction:** The object is dereferenced and eligible for garbage collection during server shutdown when the CommandManager clears its command registry.

## Internal State & Concurrency
- **State:** This class is effectively immutable after construction. Its internal state, primarily the list of subcommands inherited from AbstractCommandCollection, is populated exclusively within its constructor. No public methods exist to modify this collection post-instantiation.
- **Thread Safety:** The class is inherently **Thread-Safe**. Due to its immutable nature, multiple player threads can safely access and traverse this command node concurrently without risk of race conditions or state corruption. No synchronization or locking mechanisms are required.

## API Surface
The public contract is minimal and focused entirely on its construction-time setup. All operational logic is handled through the inherited, protected methods of AbstractCommandCollection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ParticleCommand() | Constructor | O(N) | Constructs the command group. Registers N subcommands, in this case only ParticleSpawnCommand. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by feature developers. It is part of the command system's infrastructure and is registered automatically during server initialization. The only interaction is through the server console or a client chat window.

```java
// Correct usage is by a central CommandManager during server startup
// This is a hypothetical example of the registration process.
CommandManager manager = server.getService(CommandManager.class);
manager.registerCommand(new ParticleCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create instances of ParticleCommand within game logic. The command system relies on a single, statically registered instance. Creating new instances will have no effect and wastes memory.
- **Post-Construction Modification:** Do not attempt to retrieve this object from the command registry to add more subcommands at runtime. The command hierarchy is designed to be static after the server has started.

## Data Pipeline
ParticleCommand acts as a routing step in the server's command processing pipeline. It receives a partially parsed command and dispatches it to the correct subcommand for final execution.

> Flow:
> Player Chat Input -> Network Packet -> Server Command Parser -> **ParticleCommand** (Routing Node) -> ParticleSpawnCommand (Execution Logic) -> World State Update

