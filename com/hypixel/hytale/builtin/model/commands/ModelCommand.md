---
description: Architectural reference for ModelCommand
---

# ModelCommand

**Package:** com.hypixel.hytale.builtin.model.commands
**Type:** Handler

## Definition
```java
// Signature
public class ModelCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The ModelCommand class is a server-side command handler that serves as the primary user-facing entry point for manipulating an entity's visual appearance. It translates structured text commands, entered by players or administrators, into direct modifications of the Entity Component System (ECS).

Architecturally, this class acts as a crucial bridge between three core systems:
1.  **Command System:** It registers the primary *model* command and its subcommands (*set*, *reset*), defining their syntax, arguments, and permissions.
2.  **Entity Component System (ECS):** Its primary function is to locate a target player entity and modify its **ModelComponent**. This component dictates the visual asset used to render the entity in the world.
3.  **World Threading Model:** It correctly dispatches state-mutating operations to the appropriate world thread, ensuring data consistency and preventing race conditions.

The class is structured as a root command with nested static classes for subcommands and usage variants. This pattern keeps the logic for related commands encapsulated within a single file, improving maintainability.

## Lifecycle & Ownership
-   **Creation:** A single instance of ModelCommand is created by the server's command registration service during the server bootstrap sequence. The constructor is responsible for registering all subcommands and argument variants.
-   **Scope:** The instance persists for the entire lifetime of the server. It is a stateless handler; its methods are invoked in response to player command events, but it does not retain state between invocations.
-   **Destruction:** The object is de-referenced and eligible for garbage collection when the server shuts down or the command system is reloaded.

## Internal State & Concurrency
-   **State:** The ModelCommand instance is effectively **immutable** after its construction. All fields defining its structure (subcommands, arguments) are final. The logic within its methods operates on the mutable state of the game world (the EntityStore), not on the instance itself.

-   **Thread Safety:** This class is designed to be **thread-safe**, but this is achieved through a strict adherence to the engine's concurrency patterns.
    -   Command execution may be initiated from a network thread.
    -   **WARNING:** Direct modification of an entity's components (e.g., using `store.putComponent`) from an arbitrary thread will corrupt game state.
    -   The implementation correctly uses `world.execute(() -> { ... })` to schedule all state-mutating work onto the specific world thread that owns the target entity. This is a critical pattern that guarantees safe, sequential access to ECS data. Any modifications to this class must preserve this thread-dispatching behavior.

## API Surface
The public contract is not a traditional method API but rather the command syntax it registers with the server.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| model | Command | O(1) | Opens the model selection UI for the command executor. |
| model `<player>` | Command | O(log N) | Opens the model selection UI for the specified target player. Complexity depends on player lookup. |
| model set `<model> [scale]` | Command | O(1) | Sets the executor's model to the specified ModelAsset. Optionally applies a scale. |
| model set `<model> <player> [scale]` | Command | O(log N) | Sets the target player's model. |
| model reset `[scale]` | Command | O(1) | Resets the executor's model to their default player skin. |
| model reset `<player> [scale]` | Command | O(log N) | Resets the target player's model to their default skin. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is invoked automatically by the server's command processing system when a player or the console executes the `/model` command. The system parses the input, resolves arguments, and calls the appropriate `execute` or `executeSync` method with a populated CommandContext.

```java
// This code is conceptual, representing how the command system invokes the handler.
// A developer would NOT write this.

// 1. Player types "/model set hytale:deer @p"
// 2. Command system parses this and finds the ModelSetOtherCommand handler.
// 3. It populates a CommandContext.
CommandContext context = createFromPlayerInput("/model set hytale:deer @p");

// 4. It finds the registered ModelCommand instance and invokes the subcommand.
// (This is a simplified representation)
ModelCommand.ModelSetCommand.ModelSetOtherCommand handler = findHandlerFor(context);
handler.executeSync(context);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ModelCommand()`. The command system handles the lifecycle. Creating a new instance will result in a non-functional command that is not registered with the server.
-   **Bypassing the World Scheduler:** Modifying an entity's components directly from the `executeSync` method without dispatching the work via `world.execute` is a severe concurrency violation. This will lead to race conditions, data corruption, and server instability.

    ```java
    // DANGEROUS: Do not do this.
    // This code modifies the EntityStore from a potentially non-world thread.
    @Override
    protected void executeSync(@Nonnull CommandContext context) {
        Ref<EntityStore> ref = ...;
        Store<EntityStore> store = ...;
        Model model = ...;

        // ANTI-PATTERN: This write operation is not thread-safe.
        store.putComponent(ref, ModelComponent.getComponentType(), new ModelComponent(model));
    }
    ```

## Data Pipeline
The flow of data for a successful command execution follows a clear path from user input to world state modification and client-side feedback.

> Flow:
> Player Chat Input (`/model...`) -> Server Network Layer -> Command Parser -> **ModelCommand** Execution -> World Thread Scheduler (`world.execute`) -> EntityStore State Change (`store.putComponent`) -> Component Network Sync -> Client Receives Update -> Client-Side Model Render

---

