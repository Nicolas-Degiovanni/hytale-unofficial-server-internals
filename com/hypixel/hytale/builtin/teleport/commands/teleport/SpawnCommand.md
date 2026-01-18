---
description: Architectural reference for SpawnCommand
---

# SpawnCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.teleport
**Type:** Handler

## Definition
```java
// Signature
public class SpawnCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts

The SpawnCommand class is a server-side implementation of the Command Pattern, responsible for handling the `/spawn` chat command. It serves as a high-level entry point that translates player or console input into a concrete action within the server's Entity-Component-System (ECS) architecture.

Its primary architectural role is to act as a **command-to-component bridge**. Rather than directly manipulating an entity's position, which would be a brittle and unsafe operation, SpawnCommand orchestrates a teleport by creating and attaching a `Teleport` component to the target player entity. A separate, dedicated system within the game engine is responsible for detecting and processing this component during the game tick, thereby executing the actual teleportation logic. This decouples the user-facing command interface from the complex, low-level mechanics of entity movement and world state transitions.

This class also demonstrates a composite command structure. It handles the base case of a player teleporting themselves, while also registering a nested `SpawnOtherCommand` class to manage the variant where a privileged user teleports another player. This pattern keeps related command logic encapsulated within a single file while adhering to distinct permission requirements.

Furthermore, SpawnCommand integrates with the `TeleportHistory` component, pushing the player's current location to a stack before initiating the teleport. This is a foundational piece for related features, such as a `/back` command, which would pop from this history.

## Lifecycle & Ownership

-   **Creation:** An instance of SpawnCommand is created by the server's command registration system during the server bootstrap sequence. It is not intended for manual instantiation.
-   **Scope:** Application-scoped. A single instance exists for the entire lifecycle of the running server.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the central command registry is cleared.

## Internal State & Concurrency

-   **State:** SpawnCommand is effectively stateless. It contains final fields like `spawnIndexArg` that define command argument structures, but these are immutable configurations. All state required for execution (e.g., the player, the world, command arguments) is passed into the `execute` method at runtime.
-   **Thread Safety:** The class is thread-safe due to its stateless nature and adherence to world-specific execution contexts. The `SpawnOtherCommand` variant demonstrates the critical concurrency pattern for a multi-world server: any modification to an entity's components **must** be scheduled on that entity's world thread. The use of `world.execute(...)` ensures that the addition of the `Teleport` and `TeleportHistory` components is performed safely within the target world's update loop, preventing race conditions and data corruption.

## API Surface

The primary public contract of this class is its registration with the command system, not direct method invocation. The `execute` method is the core logic handler called by the framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Framework-invoked method. Resolves the target spawn point and attaches a `Teleport` component to the executing player's entity. |

## Integration Patterns

### Standard Usage

Interaction with this class is performed exclusively through the server's command line interface by an authorized player or the server console. The command system parses the input and routes it to the appropriate handler.

```
// Player-issued command in chat
/spawn

// Admin-issued command to teleport another player
/spawn Notch

// Admin-issued command to teleport another player to a specific spawn index
/spawn Notch 2
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new SpawnCommand()`. The command system is solely responsible for the lifecycle of command objects. Manual creation will result in a non-functional command that is not registered with the server.
-   **Direct Invocation:** Never call the `execute` method directly. Doing so bypasses the entire command processing pipeline, including critical permission checks, argument parsing, and context setup. This will lead to `NullPointerException`s and potential security vulnerabilities.
-   **Assuming Synchronous Teleport:** The command only *requests* a teleport by adding a component. The action is asynchronous and will be completed later in the server tick by a different system. Any code that executes immediately after a command is dispatched cannot assume the player has already been moved.

## Data Pipeline

The SpawnCommand initiates a data flow that begins with user input and results in a change to the player's state, which is then synchronized back to the client.

> Flow:
> Player Chat Input (`/spawn`) -> Network Packet -> Server Command Parser -> **SpawnCommand.execute** -> ECS `Store.addComponent(Teleport)` -> Teleport Processing System -> Entity `TransformComponent` Update -> World State Synchronization -> Network Packet -> Client-side Entity Position Update

