---
description: Architectural reference for TeleportHomeCommand
---

# TeleportHomeCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.teleport
**Type:** Transient

## Definition
```java
// Signature
public class TeleportHomeCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts

The TeleportHomeCommand class is a concrete implementation of the Command Pattern, designed to handle the server-side logic for the player-facing `/home` command. It acts as a translator, converting a simple text-based intention from a player into a series of state changes within the Entity Component System (ECS).

Architecturally, its primary role is to decouple the command input system from the core gameplay mechanics of teleportation. It does not contain the logic for moving a player. Instead, it orchestrates the process by:

1.  **Recording State:** Capturing the player's current location to a TeleportHistory component for the `/back` command.
2.  **Resolving Destination:** Querying the player's data to find their designated respawn (home) position.
3.  **Declaring Intent:** Adding a transient **Teleport** component to the player entity. This component acts as a request or a "flag" for other systems.

The actual teleportation is handled by a separate, dedicated system (e.g., a TeleportSystem) that processes entities with the Teleport component each tick. This separation of concerns is a fundamental ECS principle, ensuring that command handlers remain lightweight and focused solely on intent, while complex state manipulation is managed by dedicated systems.

## Lifecycle & Ownership

-   **Creation:** A single instance of TeleportHomeCommand is instantiated by the server's command registry during the bootstrap phase. The system scans for command classes and registers them for the server's lifetime.
-   **Scope:** The object instance is a long-lived singleton managed by the command registry. However, its execution context is transient; the *execute* method is invoked for a brief moment to handle a specific command request and does not persist state between calls.
-   **Destruction:** The instance is de-referenced and eligible for garbage collection only when the server shuts down or the command module is unloaded.

## Internal State & Concurrency

-   **State:** This class is effectively stateless and immutable after construction. It contains no instance fields that change during its lifecycle. All necessary state (player reference, world data) is passed as arguments to the *execute* method. The static message field is a compile-time constant.

-   **Thread Safety:** The object itself is thread-safe due to its stateless nature. The Hytale server architecture guarantees that the *execute* method is invoked on the appropriate world thread, with safe access to the entity components being modified. Direct, multi-threaded invocation of the *execute* method from outside the engine's command processor is unsupported and would lead to severe concurrency violations.

## API Surface

The public contract is minimal, intended for framework integration rather than direct developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TeleportHomeCommand() | Constructor | O(1) | Registers the command name, description, and required permission with the base command system. |
| execute(...) | protected void | O(1) | Framework-invoked method. Executes the command logic. Throws assertions if the player entity is in an invalid state (e.g., missing TransformComponent). |

## Integration Patterns

### Standard Usage

This class is not intended to be used directly by developers. It is automatically discovered and managed by the server's command processing system. A player triggers its execution by typing the command in chat.

```
// This class is invoked by the engine.
// A player in-game types:
/home
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new TeleportHomeCommand()`. The command will not be registered with the server and will have no effect. The framework handles instantiation and registration.

-   **Manual Invocation:** Calling the *execute* method directly is a critical error. This bypasses the entire command processing pipeline, including crucial permission checks, argument parsing, and context setup. Manually constructing the required `Store`, `Ref`, and `World` objects is complex and risks corrupting world state.

## Data Pipeline

The execution of this command initiates a clear, one-way data flow that transforms a player's request into a change in the game world.

> Flow:
> Player Chat Input (`/home`) -> Network Layer -> Command Parser -> **TeleportHomeCommand.execute()** -> ECS State Change (adds TeleportHistory & Teleport components) -> TeleportSystem (reads Teleport component) -> Entity Transform Update -> Network Packet -> Client Position Update

