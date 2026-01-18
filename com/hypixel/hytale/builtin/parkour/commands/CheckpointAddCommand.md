---
description: Architectural reference for CheckpointAddCommand
---

# CheckpointAddCommand

**Package:** com.hypixel.hytale.builtin.parkour.commands
**Type:** Singleton Command Handler

## Definition
```java
// Signature
public class CheckpointAddCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The CheckpointAddCommand class is a server-side command handler responsible for creating new parkour checkpoints in the world. It functions as a direct bridge between player input (via the chat command system) and the server's core Entity Component System (ECS).

Architecturally, this class is a *World State Mutator*. Its sole purpose is to receive a command, validate the input against the global state managed by the ParkourPlugin, and if valid, instantiate a new, fully configured entity representing the checkpoint. This entity is composed of several components that define its behavior, appearance, and persistence.

This command handler is a leaf node in the server architecture; it orchestrates calls to the ECS and other services but does not expose any API for other systems to consume. It relies on the ParkourPlugin singleton to act as a service locator for shared configuration (the checkpoint model) and state (the map of existing checkpoint UUIDs).

### Lifecycle & Ownership
- **Creation:** A single instance of CheckpointAddCommand is created and registered with the server's CommandSystem when the parent ParkourPlugin is initialized.
- **Scope:** The instance persists for the entire lifecycle of the ParkourPlugin. It is a stateless singleton whose execute method is invoked upon matching player input.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the ParkourPlugin is disabled or the server shuts down.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. The member field `indexArg` is a final, immutable definition for argument parsing. All state that is read or modified—such as the player's transform, the world's EntityStore, and the ParkourPlugin's checkpoint map—is external to this class.

- **Thread Safety:** **This class is not thread-safe.** The `execute` method performs direct mutations on the world's EntityStore. All interactions with the ECS and game state must be performed on the main server thread. Invoking this logic from any other thread will lead to world corruption, race conditions, and server instability.

## API Surface
The public contract is defined by its inheritance from AbstractPlayerCommand and is intended for consumption by the CommandSystem only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Creates a new parkour checkpoint entity at the player's location. Throws an assertion error if the player's TransformComponent is missing. |

## Integration Patterns

### Standard Usage
This class is not designed for direct invocation by game logic developers. The server's CommandSystem invokes the `execute` method when a player with appropriate permissions runs the corresponding command. The system provides the necessary context and world state.

The following conceptual example illustrates how the system might be registered by its parent plugin.

```java
// Within the ParkourPlugin's initialization logic
CommandSystem commandSystem = server.getCommandSystem();
commandSystem.getRootNode("checkpoint")
    .addChild(new CheckpointAddCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new CheckpointAddCommand()` in your own logic. The command will not be registered with the server and will have no effect. Registration is the responsibility of the parent plugin.
- **Direct Invocation:** Never call the `execute` method directly. It requires a precisely constructed `CommandContext` and valid ECS references that are only guaranteed to be correct when provided by the CommandSystem during a live command execution.
- **State Management:** Do not attempt to add stateful member variables to this class. Command handlers are expected to be stateless singletons. All state should be passed in via the `execute` method parameters or retrieved from appropriate world services.

## Data Pipeline
The execution of this command initiates a one-way data flow from player input to a permanent world state change, which is then broadcast to clients.

> Flow:
> Player Chat Input (`/checkpoint add 1`) → Server Network Layer → Command Parser → **CheckpointAddCommand.execute()** → EntityStore.addEntity() → World State Mutation → Network Replication Layer → Client Render Update

