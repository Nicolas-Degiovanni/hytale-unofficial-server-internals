---
description: Architectural reference for WorldConfigSetPvpCommand
---

# WorldConfigSetPvpCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.worldconfig
**Type:** Transient

## Definition
```java
// Signature
public class WorldConfigSetPvpCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WorldConfigSetPvpCommand is a concrete implementation of the Command Pattern, designed to modify a specific server world configuration: the Player-vs-Player (PVP) enabled state. It resides within the server's command processing framework and is specialized to operate on a World object.

As a subclass of AbstractWorldCommand, it integrates directly with the command system's world-management context. The framework guarantees that when this command is executed, it will be provided with a valid World instance and its corresponding EntityStore.

Its sole responsibility is to:
1.  Parse a boolean argument from the command context.
2.  Mutate the state of the target World's WorldConfig object.
3.  Signal that the configuration has changed via markChanged.
4.  Provide feedback to the command issuer.

This class decouples the command invocation logic (parsing, permissions) from the business logic of changing a world setting.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's command registration system during the server bootstrap phase. A single instance is typically created and registered under the name "pvp" within its parent command group.
-   **Scope:** The object instance persists for the entire server session. However, its execution is scoped to a single, atomic command invocation. It holds no state between executions.
-   **Destruction:** The instance is destroyed when the server shuts down and the command registry is cleared from memory.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. Its only field, stateArg, is a final definition of a command argument and is initialized once in the constructor. All state required for execution (the world, the command context) is passed as parameters to the execute method.
-   **Thread Safety:** The object itself is thread-safe due to its immutable and stateless design. However, the execute method performs a write operation on a WorldConfig object. The overall thread safety of the operation is therefore dependent on the concurrency guarantees of the World and WorldConfig classes. It is assumed that the command system serializes command execution on a per-world basis to prevent race conditions when modifying world state.

## API Surface
The primary contract is the protected execute method, which is invoked by the command system framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Modifies the PVP state on the provided World's WorldConfig. Sends a confirmation message to the command source. |

## Integration Patterns

### Standard Usage
This class is not intended for direct developer interaction. It is invoked automatically by the server's command handler in response to player or console input.

An administrator would trigger this command in-game:
```
/world config pvp true
```
Or to toggle the current state:
```
/world config pvp
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WorldConfigSetPvpCommand()`. Command objects must be registered with and managed by the command system to function correctly.
-   **Manual Execution:** Do not call the `execute` method directly. Doing so bypasses critical infrastructure such as permission checks, argument parsing, and context provision, leading to unpredictable behavior and likely NullPointerExceptions.

## Data Pipeline
The flow of data for a successful command execution is linear and transactional.

> Flow:
> Player Chat Input -> Network Packet -> Server Command Parser -> Command Dispatcher -> **WorldConfigSetPvpCommand.execute()** -> WorldConfig.setPvpEnabled() -> WorldConfig.markChanged() -> World Persistence System

Simultaneously, a feedback message is generated and sent back to the source.

> Feedback Flow:
> **WorldConfigSetPvpCommand.execute()** -> CommandContext.sendMessage() -> Message Translation System -> Network Layer -> Client Chat UI

