---
description: Architectural reference for PlayerEffectClearCommand
---

# PlayerEffectClearCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.effect
**Type:** Transient

## Definition
```java
// Signature
public class PlayerEffectClearCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The PlayerEffectClearCommand is a command handler within the server's Command System. It serves as a specific endpoint that translates a user-issued command into a direct action within the Entity Component System (ECS). Its primary architectural role is to act as a controller that mutates the state of a player entity.

This class implements two distinct command variants using a nested class pattern:
1.  **Self-Targeting:** The primary class handles the `/effect clear` command, where the command issuer is also the target. It inherits from AbstractPlayerCommand, which provides a simplified execution contract with direct access to the player's entity components.
2.  **Other-Targeting:** The nested PlayerEffectClearOtherCommand handles `/effect clear <player>`, targeting another player. This variant demonstrates a more complex interaction pattern, requiring explicit lookup of the target player's entity reference, their associated World instance, and careful thread synchronization to safely modify their state.

This command is a terminal node in the command processing pipeline. It does not delegate to other systems beyond the ECS; its sole responsibility is to locate the target's EffectControllerComponent and invoke its clearEffects method.

## Lifecycle & Ownership
-   **Creation:** A single instance of PlayerEffectClearCommand is instantiated by the server's command registration service during the server bootstrap sequence. The system scans for command definitions and populates a central command registry.
-   **Scope:** The command object is a stateless singleton that persists for the entire server runtime. It is held within the central command registry and is never destroyed until server shutdown.
-   **Destruction:** The object is eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is stateless and immutable. Its fields consist of final Message constants and argument definitions initialized at construction. It does not cache data or maintain any state between invocations. All necessary state is provided via the CommandContext and method parameters during execution.
-   **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, the operations it performs on the ECS are not. The design correctly addresses this:
    -   The base class `execute` method is invoked by the command system on the source player's main world thread, ensuring safe access to their own components.
    -   The nested PlayerEffectClearOtherCommand's `executeSync` method correctly retrieves the *target* player's World instance and schedules the entity modification logic using `world.execute`. This is a critical pattern to prevent race conditions and ensure all ECS mutations occur on the appropriate thread for that world.

    **WARNING:** Failure to use the `world.execute` pattern when modifying entities in other worlds will lead to severe concurrency issues, data corruption, and server instability.

## API Surface
The public contract is implicitly defined by the command system's invocation of the protected `execute` and `executeSync` methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(1) | Executes the command logic for the self-targeting variant. Assumes execution on the correct world thread. |
| executeSync(...) | protected void | O(log N) | Executes the command logic for the other-targeting variant. Involves player lookup and schedules the final action on the target's world thread. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. The command system parses player input, identifies the command, and invokes the appropriate `execute` method with a fully populated CommandContext.

The primary integration point is registering the command with the system.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PlayerEffectClearCommand()`. The command will not be registered with the server and will be unusable.
-   **Direct Invocation:** Never call the `execute` or `executeSync` methods directly. Doing so bypasses the command system's critical infrastructure, including permission checks, argument parsing, and thread management.
-   **Cross-Thread ECS Modification:** Do not attempt to get a player's components and modify them from the `executeSync` method without wrapping the logic in `world.execute`. This is the most common and dangerous error when working with multi-world command logic.

## Data Pipeline
The class acts as a processor in the server's command-to-action data pipeline.

> **Flow (Self-Target):**
> Player Chat Input -> Network Layer -> Command Parser -> **PlayerEffectClearCommand.execute** -> EffectControllerComponent.clearEffects -> Player State Update

> **Flow (Other-Target):**
> Player Chat Input -> Network Layer -> Command Parser -> **PlayerEffectClearOtherCommand.executeSync** -> PlayerRef Lookup -> World.execute(lambda) -> EffectControllerComponent.clearEffects -> Target Player State Update

