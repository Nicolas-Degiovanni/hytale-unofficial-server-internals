---
description: Architectural reference for PrefabPathNewCommand
---

# PrefabPathNewCommand

**Package:** com.hypixel.hytale.builtin.path.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabPathNewCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The PrefabPathNewCommand class is a *Command Object* that serves as the entry point for creating a new NPC path prefab. It is part of the server's command processing framework and is specifically designed to be invoked by a player with appropriate permissions.

Its primary architectural role is to act as a translator between raw player input and the core game systems. It does not contain the complex logic for path creation itself. Instead, it performs three critical functions:
1.  **Argument Parsing:** It defines and parses the required and optional arguments for the command, such as the path name, pause time, and observation angle.
2.  **State Orchestration:** It initiates the creation of the path's first marker by delegating to the PrefabPathHelper utility. This follows the Single Responsibility Principle, separating command handling from business logic.
3.  **Session Management:** It interacts with the BuilderToolsPlugin to update the executing player's session state, marking the new path as the "active" one for subsequent editing commands.

This class is a thin, stateless bridge between the command system and the underlying world modification and player state services.

## Lifecycle & Ownership
-   **Creation:** A single instance of PrefabPathNewCommand is instantiated by the server's CommandSystem during the plugin loading phase. It is discovered through reflection or explicit registration and added to a central command registry.
-   **Scope:** The object instance persists for the entire server session. It is designed to be stateless, allowing the single instance to be reused for every execution of the command by any player.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when its parent plugin is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** This class is effectively **immutable** after its initial construction. The fields defining the command arguments are configured once by the constructor and are never modified thereafter. All execution-specific data is passed into the *execute* method via its parameters, ensuring no state is stored on the instance itself between calls.

-   **Thread Safety:** This class is **not thread-safe** and is designed to be executed exclusively on the main server thread. The *execute* method directly manipulates world state via the EntityStore and player components. Any attempt to invoke it from an asynchronous thread will lead to race conditions, data corruption, and server instability. The command framework guarantees serialized, on-thread execution.

## API Surface
The public contract is minimal, consisting of the constructor for framework instantiation and the overridden *execute* method for command logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Parses arguments from the context and orchestrates the creation of a new path. Modifies world and player state. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is invoked by the server's command handler when a player types the corresponding command into the chat.

```java
// This class is not invoked directly by developers.
// It is registered with the CommandSystem and triggered by player chat input.

// Example of the player command that triggers this class:
// /npc path new "My Cool Path" 5.0 90.0
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PrefabPathNewCommand()` in your code. The class must be registered with the server's command system to receive the necessary context for execution.
-   **Manual Execution:** Calling the *execute* method manually is a critical error. This bypasses the command system's permission checks, argument parsing pipeline, and context injection, which will result in unpredictable behavior and likely throw a NullPointerException.

## Data Pipeline
The flow of data for this command begins with player input and results in a modification of both world state and the player's session state.

> Flow:
> Player Chat Input -> Server Command Parser -> **PrefabPathNewCommand.execute()** -> PrefabPathHelper -> World State Update (EntityStore) & Player State Update (BuilderToolsPlugin) -> Feedback Message to Player

