---
description: Architectural reference for HubCommand
---

# HubCommand

**Package:** com.hypixel.hytale.builtin.creativehub.command
**Type:** Transient

## Definition
```java
// Signature
public class HubCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The HubCommand class serves as a user-facing entry point for the Creative Hub system. It is a concrete implementation of the server's command pattern, responsible for handling the `/hub` command and its aliases.

Architecturally, this class acts as a **Controller** or **Facade** for a complex player teleportation workflow. It does not contain business logic itself; instead, it orchestrates interactions between several core server systems:
- **Command System:** Registers the command and invokes its execution logic.
- **Universe & World Management:** Queries the current world state and searches for the designated parent hub world.
- **Entity Component System (ECS):** Reads configuration data attached to entities, specifically the CreativeHubEntityConfig, to determine context.
- **CreativeHubPlugin:** The primary service for managing hub instances. HubCommand delegates the responsibility of finding or creating a hub world instance to this plugin.
- **InstancesPlugin:** The underlying service responsible for the low-level mechanics of teleporting a player entity between different world instances.

Its primary function is to validate the player's request, resolve the target hub world based on the player's current context, and initiate a teleportation sequence.

## Lifecycle & Ownership
- **Creation:** A single instance of HubCommand is created by the server's command registration system during the bootstrap phase of the CreativeHubPlugin. It is not instantiated per-command execution.
- **Scope:** The object is a long-lived singleton managed by the command system. Its lifecycle is tied directly to the lifecycle of the plugin that registers it. It persists for the entire server session.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the CreativeHubPlugin is unloaded or the server shuts down, at which point the command registry is cleared.

## Internal State & Concurrency
- **State:** HubCommand is **stateless**. It contains no mutable instance fields. All necessary state for an operation (e.g., the player, the world, the entity store) is passed as arguments to the `execute` method. The static Message fields are immutable and thread-safe by definition.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, the `execute` method is designed to be called exclusively from the main server thread that manages the corresponding world. The methods it invokes on other systems like Universe, World, and various plugins are **not guaranteed to be thread-safe** and must be called from the correct game loop context. Concurrently invoking `execute` from multiple threads would lead to race conditions and severe state corruption in the underlying world systems.

## API Surface
The public API is minimal and dictated by its parent class, AbstractPlayerCommand. The primary entry point is the `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| HubCommand() | constructor | O(1) | Initializes the command name, description, aliases, and required permission group. Intended for framework use only. |
| execute(...) | void | O(N) | Orchestrates the hub teleportation logic. Complexity is dependent on world lookups and instance spawning, which can be non-trivial. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. The server's command dispatcher is the sole intended caller. A player triggers its execution by typing a registered command in the chat console.

```
// This logic is handled internally by the server's command dispatcher.
// A developer would never write this code.

// 1. Player types "/hub" in chat.
// 2. Server parses the command and finds the registered HubCommand instance.
// 3. Server populates a CommandContext and invokes the execute method.
command.execute(context, store, ref, playerRef, world);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new HubCommand()`. The command must be registered with and managed by the server's command system to function correctly.
- **Manual Invocation:** Do not call the `execute` method directly from other plugin code. This bypasses critical infrastructure, including permission checks and context validation. To teleport a player, use the appropriate high-level API, such as InstancesPlugin.teleportPlayerToInstance.

## Data Pipeline
The HubCommand initiates a data flow that transforms a player command into a change in the player's world state.

> Flow:
> Player Chat Input (`/hub`) -> Network Ingress -> Command Dispatcher -> **HubCommand.execute()** -> Universe.getWorld() -> CreativeHubPlugin.getOrSpawnHubInstance() -> InstancesPlugin.teleportPlayerToInstance() -> Player Entity State Update -> Network Egress (Teleport Packet) -> Client World Transition

