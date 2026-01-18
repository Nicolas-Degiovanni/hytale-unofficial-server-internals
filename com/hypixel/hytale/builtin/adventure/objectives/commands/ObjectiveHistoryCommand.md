---
description: Architectural reference for ObjectiveHistoryCommand
---

# ObjectiveHistoryCommand

**Package:** com.hypixel.hytale.builtin.adventure.objectives.commands
**Type:** Transient

## Definition
```java
// Signature
public class ObjectiveHistoryCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The ObjectiveHistoryCommand is a server-side, player-executable command that functions as a read-only interface to the Adventure Mode objective system. It provides players with a mechanism, typically via a chat command, to view their own completed objectives and objective lines.

Architecturally, this class serves as a terminal node in the server's command processing pipeline. It bridges user input directly to data retrieval from the Entity Component System (ECS). By extending AbstractPlayerCommand, it integrates seamlessly with the server's command registry and ensures that it is always executed within the context of a specific player. This inheritance provides the `execute` method with the necessary player entity reference (`PlayerRef`) and a handle to the world's primary data `Store`, which are required to query for the relevant components.

This command is fundamentally a data presentation tool; it performs no state modification and its sole responsibility is to query an entity's ObjectiveHistoryComponent, format the contained data into a human-readable string, and transmit it back to the originating player.

### Lifecycle & Ownership
- **Creation:** An instance of ObjectiveHistoryCommand is created on-demand by the server's core command system. This occurs when a player sends a command string that the system resolves to this class. The instantiation is managed entirely by the engine's command dispatcher.
- **Scope:** The object's lifetime is exceptionally brief, scoped precisely to the execution of a single command. It is created, its `execute` method is invoked once, and it is then immediately eligible for garbage collection.
- **Destruction:** The object is destroyed by the Java Garbage Collector shortly after the `execute` method returns. It holds no persistent state and is not referenced by any long-lived system.

## Internal State & Concurrency
- **State:** ObjectiveHistoryCommand is a stateless class. It contains no instance fields for storing data. All information required for its operation is provided as method-local parameters to the `execute` function by the command system.
- **Thread Safety:** This class is inherently thread-safe due to its stateless design. However, the execution of the command is expected to occur on the main server thread for the corresponding world. The underlying ECS `Store` is responsible for managing its own concurrency and ensuring safe access to component data. Direct, multi-threaded invocation of the `execute` method is an unsupported and dangerous pattern.

## API Surface
The public contract is defined by its constructor and the overridden `execute` method, which are invoked exclusively by the server's command infrastructure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ObjectiveHistoryCommand() | constructor | O(1) | Instantiates the command and registers its name and permission key. |
| execute(context, store, ref, playerRef, world) | void | O(N+M) | Fetches, formats, and sends the player's objective history. Complexity is linear based on the number of completed objectives (N) and objective lines (M). Throws AssertionError if the player entity does not have an ObjectiveHistoryComponent. |

## Integration Patterns

### Standard Usage
This class is not designed for direct programmatic invocation by developers. It is a system-level component automatically discovered and managed by the server's command handler. The standard interaction is a player executing the command in-game.

A developer's interaction with this system would typically be limited to ensuring the player entity has the required `ObjectiveHistoryComponent` attached.

```java
// A player entity must have this component for the command to succeed.
// This code would exist in a system that manages player setup.

EntityRef playerEntity = ...;
ObjectiveHistoryComponent history = new ObjectiveHistoryComponent();

// The component is attached to the player entity.
// The ObjectiveHistoryCommand can now read from it.
entityStore.setComponent(playerEntity, history);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Invocation:** Manually creating an instance via `new ObjectiveHistoryCommand()` and calling `execute` is a severe anti-pattern. This bypasses the entire command system infrastructure, including context setup, permission validation, and thread management, leading to unpredictable behavior and likely NullPointerExceptions.
- **State Caching:** Adding member variables to this class to cache data is incorrect. Each command execution is handled by a new, ephemeral instance. Any state must be stored in a persistent location, such as an ECS component.

## Data Pipeline
The flow of data for this command begins with player input and terminates with a message displayed on the player's screen. The ObjectiveHistoryCommand is a critical processing step in the middle of this flow.

> Flow:
> Player Command Input -> Network Packet -> Server Command System -> **ObjectiveHistoryCommand.execute()** -> ECS Store Read (ObjectiveHistoryComponent) -> Formatted String -> Message -> Network Packet -> Client Chat UI

