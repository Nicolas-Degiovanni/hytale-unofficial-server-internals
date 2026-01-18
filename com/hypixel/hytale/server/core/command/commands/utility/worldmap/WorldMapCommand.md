---
description: Architectural reference for WorldMapCommand
---

# WorldMapCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.worldmap
**Type:** Transient

## Definition
```java
// Signature
public class WorldMapCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The WorldMapCommand class is a structural component within the server's command handling system. It embodies the **Composite Pattern**, acting as a non-terminal node or a "collection" in the command tree. Its primary function is not to execute logic, but to group related sub-commands under a single, user-facing namespace: *worldmap*.

By extending AbstractCommandCollection, it inherits the necessary infrastructure to register a primary command name, aliases, and a list of child command objects. When a player executes a command like `/worldmap reload`, the command dispatcher first resolves the root `worldmap` token to this collection. It then delegates parsing of the subsequent token, `reload`, to one of its registered children, in this case the WorldMapReloadCommand.

This architectural choice promotes modularity and discoverability. It allows developers to organize complex, multi-part commands into a clean hierarchy without polluting the global command namespace.

## Lifecycle & Ownership
- **Creation:** A new instance of WorldMapCommand is created by the server's central CommandRegistry during the server startup or plugin loading phase. The registry typically discovers command classes via reflection or explicit registration and instantiates them.
- **Scope:** The object's lifespan is ephemeral. It exists primarily during the command registration process. Once its constructor has completed and its sub-commands have been added to the internal collection, its configuration is consumed by the CommandRegistry to build the final, immutable command tree.
- **Destruction:** After the server's command tree is built, this specific container object is no longer actively referenced by the core command dispatcher and becomes eligible for garbage collection. The command definitions it provided persist within the registry, but the instance itself is discarded.

## Internal State & Concurrency
- **State:** The state of this object is configured exclusively within its constructor and is considered **effectively immutable** post-construction. The internal list of sub-commands is not designed to be modified at runtime.
- **Thread Safety:** This class is **not thread-safe**. All instantiation and configuration are expected to occur synchronously on the main server thread during the initial bootstrap sequence. It is a severe design violation to attempt to modify this object from another thread or after the server has finished loading.

## API Surface
This class has no meaningful public API surface beyond its constructor. All interaction is declarative and handled by the command registration system. The methods used for configuration, such as addSubCommand, are inherited from its parent and are intended for use only within the constructor.

## Integration Patterns

### Standard Usage
Developers do not interact with this class programmatically. Its integration is purely declarative. To add a new sub-command to the `/worldmap` command, a developer would create a new command class and add an instantiation of it to the constructor of this class.

```java
// Example: Adding a hypothetical "WorldMapSetMarkerCommand"
public class WorldMapCommand extends AbstractCommandCollection {
   public WorldMapCommand() {
      super("worldmap", "server.commands.worldmap.desc");
      this.addAliases("map");
      this.addSubCommand(new WorldMapReloadCommand());
      this.addSubCommand(new WorldMapDiscoverCommand());
      this.addSubCommand(new WorldMapUndiscoverCommand());
      this.addSubCommand(new WorldMapClearMarkersCommand());
      this.addSubCommand(new WorldMapViewRadiusSubCommand());
      
      // Add the new command here
      this.addSubCommand(new WorldMapSetMarkerCommand());
   }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of WorldMapCommand manually via `new WorldMapCommand()`. It will be a disconnected object that is not registered with the server's command system and will have no effect.
- **Post-Construction Modification:** Do not attempt to get a reference to a registered command collection and call `addSubCommand` or other mutator methods after server initialization. This will lead to an inconsistent command state and is not supported.

## Data Pipeline
As a structural component, WorldMapCommand acts as a routing node in the command processing pipeline. It does not transform data but directs the flow of control.

> Flow:
> Raw Player Input (`/worldmap discover 100`) -> Network Layer -> Command Parser -> **WorldMapCommand** (Resolves `worldmap`) -> WorldMapDiscoverCommand (Receives `discover 100` for execution)

