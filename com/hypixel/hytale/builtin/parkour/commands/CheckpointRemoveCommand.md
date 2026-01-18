---
description: Architectural reference for CheckpointRemoveCommand
---

# CheckpointRemoveCommand

**Package:** com.hypixel.hytale.builtin.parkour.commands
**Type:** Transient

## Definition
```java
// Signature
public class CheckpointRemoveCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The CheckpointRemoveCommand is a server-side command handler responsible for processing the administrative action of removing a parkour checkpoint from a world. As a subclass of AbstractWorldCommand, it is intrinsically tied to the server's command processing system and operates within the context of a specific game World.

This class acts as a controller in the Model-View-Controller pattern. It receives input via the CommandContext, manipulates the model (the World's EntityStore and the ParkourPlugin's internal state), and provides feedback to the user (the view) by sending messages. Its primary function is to translate a user-issued command into a concrete world modification, ensuring the integrity of the parkour system's data structures.

It directly couples the generic command system to the specific business logic of the ParkourPlugin, retrieving the plugin's central state to identify and remove the target entity.

### Lifecycle & Ownership
- **Creation:** A single instance is created by the ParkourPlugin during its initialization phase. This instance is then registered with the server's central CommandSystem.
- **Scope:** The object's lifecycle is bound to the ParkourPlugin. It persists as long as the plugin is enabled and the command is registered. It is a stateless service object; no instance-level state is maintained between executions.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the ParkourPlugin is disabled or the server shuts down, at which point its command registration is removed.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. The field indexArg is a definition for argument parsing, not a container for execution state. All stateful data, such as the checkpoint-to-UUID mapping and the world's entities, is accessed from external systems like the ParkourPlugin and the EntityStore during the execution of the command.
- **Thread Safety:** This component is not thread-safe and is not designed for concurrent access. Command execution is expected to be serialized and performed on the main server thread. Direct calls to the execute method from other threads will lead to race conditions and world corruption, as it performs non-atomic read-modify-write operations on both the ParkourPlugin's state and the World's EntityStore.

## API Surface
The public contract is implicitly defined by its registration as a command and its base class. The core logic is encapsulated in the protected execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Executes the removal logic. Reads the required integer index from the context, looks up the entity UUID in the ParkourPlugin's map, and requests its removal from the world's EntityStore. Sends success or failure messages to the user. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is invoked by the server's command system when a player or the console executes the corresponding command.

```text
// User input in game client or server console
/checkpoint remove 1
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new CheckpointRemoveCommand()`. The command system manages the lifecycle. Manual instantiation creates an object that is not registered to handle any commands.
- **Manual Invocation:** Calling the `execute` method directly bypasses critical infrastructure such as permission checks, argument parsing, and context provision. This can lead to unpredictable behavior and server instability.
- **State Assumption:** Do not assume an entity exists just because its index is in the `checkpointUUIDMap`. The command correctly checks if the entity `Ref` is valid before attempting removal, a practice that must be maintained.

## Data Pipeline
The flow of data for a checkpoint removal operation is linear and initiated by an external agent (a player or console).

> Flow:
> User Command String -> Server Command Parser -> **CheckpointRemoveCommand.execute()** -> ParkourPlugin.getCheckpointUUIDMap() (Read) -> EntityStore.removeEntity() (Write) -> CommandContext.sendMessage() (Feedback)

