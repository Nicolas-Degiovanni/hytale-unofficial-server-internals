---
description: Architectural reference for RecipeCommand
---

# RecipeCommand

**Package:** com.hypixel.hytale.builtin.crafting.commands
**Type:** Utility

## Definition
```java
// Signature
public class RecipeCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The RecipeCommand class serves as a high-level command dispatcher, grouping all recipe-related server commands under a single `/recipe` entry point. It follows the **Composite Pattern**, acting as a collection container for more specific sub-command objects: `Learn`, `Forget`, and `List`.

Architecturally, this class is a critical bridge between the server's generic Command System and the specialized logic of the CraftingPlugin. It is not responsible for the business logic of managing recipes itself; rather, it is responsible for:
1.  Defining the command syntax, including sub-commands and their required arguments (e.g., `item`, `player`).
2.  Parsing and validating arguments provided by the command executor.
3.  Delegating the core logic to the CraftingPlugin service.
4.  Formatting and sending response messages (success or failure) back to the executor via the CommandContext.

The use of nested static classes for sub-commands (e.g., `Learn`, `LearnOther`) is a deliberate design choice to encapsulate command-specific logic and argument definitions, preventing pollution of the parent class. The distinction between `AbstractPlayerCommand` variants and `CommandBase` variants handles the separate logic paths for a command targeting the self versus targeting another player.

### Lifecycle & Ownership
- **Creation:** A single instance of RecipeCommand is created during server bootstrap. It is registered with the central CommandSystem, which then owns the reference for the server's lifetime.
- **Scope:** This object is a long-lived singleton for the entire server session. It is designed to be stateless and is reused for every `/recipe` command invocation.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only upon server shutdown, when the CommandSystem is cleared.

## Internal State & Concurrency
- **State:** RecipeCommand is **stateless and effectively immutable** post-construction. Its internal state consists of a final collection of its sub-commands, which is populated in the constructor and never modified thereafter. The class does not cache data or maintain any per-request state.

- **Thread Safety:** The RecipeCommand instance itself is thread-safe due to its stateless nature. However, the operations it initiates have strict threading requirements. Modifying a player's recipe list is a world-state mutation and **must** occur on the main world thread.

    **WARNING:** The `...Other` command variants (e.g., `LearnOther`) correctly handle this by wrapping their logic in a `world.execute(...)` call. This marshals the execution from a potentially asynchronous command-handling thread to the synchronous world-update thread, preventing race conditions and ensuring data consistency within the EntityStore.

## API Surface
The primary programmatic interaction with this class is its instantiation and registration. It is not intended to be called directly after initialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| RecipeCommand() | Constructor | O(1) | Initializes the command collection and registers the `learn`, `forget`, and `list` sub-commands. |

## Integration Patterns

### Standard Usage
This class is not used directly in game logic. It is registered with the server's command system one time during server initialization.

```java
// Example of registration during server startup
// This code would exist within the server's bootstrap sequence.

CommandSystem commandSystem = server.getCommandSystem();
commandSystem.register(new RecipeCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create new instances of RecipeCommand outside of the initial command registration process. It is designed to be a singleton managed by the CommandSystem.

- **Stateful Modifications:** Do not attempt to add or remove sub-commands from this collection after its initial construction. This would violate its stateless design and could lead to unpredictable behavior.

- **Bypassing the World Thread:** The logic for modifying player data (via CraftingPlugin) must be executed on the main world thread. The `world.execute` pattern used in the `...Other` sub-commands is mandatory for thread safety. Directly invoking this logic from another thread will corrupt player state.

## Data Pipeline
The flow of data for a typical command execution demonstrates the class's role as a dispatcher and delegator.

> Flow:
> Player Command Input (`/recipe learn ...`) -> Server Command Parser -> **RecipeCommand** -> `Learn` Sub-command -> Argument Validation -> `world.execute()` -> `CraftingPlugin.learnRecipe()` -> Player Component Mutation in EntityStore -> `CommandContext.sendMessage()` -> Player Client Feedback

