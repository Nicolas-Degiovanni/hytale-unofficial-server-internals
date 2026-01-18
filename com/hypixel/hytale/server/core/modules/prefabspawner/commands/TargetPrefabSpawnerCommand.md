---
description: Architectural reference for TargetPrefabSpawnerCommand
---

# TargetPrefabSpawnerCommand

**Package:** com.hypixel.hytale.server.core.modules.prefabspawner.commands
**Type:** Transient / Command Template

## Definition
```java
// Signature
public abstract class TargetPrefabSpawnerCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The TargetPrefabSpawnerCommand is an abstract base class that serves as a foundational template for any server command designed to interact with a PrefabSpawnerState block in the game world. It embodies the Template Method design pattern, providing a rigid, reusable algorithm for target acquisition while deferring the specific action to its subclasses.

Its primary architectural role is to decouple the logic of *finding* a spawner from the logic of *acting* on that spawner. It standardizes how commands resolve their target coordinates, supporting two primary modes:
1.  **Explicit Targeting:** A player or system can provide an explicit block position via a command argument.
2.  **Implicit Targeting:** If no position is provided, the system performs a raycast from the command sender's point of view to find the block they are looking at.

After successfully identifying and validating that the target block is indeed a PrefabSpawnerState, it delegates control to the concrete implementation's `execute` method, passing along the command context and a direct reference to the spawner's state.

## Lifecycle & Ownership
-   **Creation:** A concrete subclass instance is created once by the server's command registration system during server bootstrap. This single instance is registered and reused for all subsequent invocations of that command.
-   **Scope:** The command object itself is a long-lived singleton, persisting for the entire server session within the command registry. However, its execution is transient; the `execute` method is invoked as a new, self-contained operation each time the command is run.
-   **Destruction:** The object is marked for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless from an execution perspective. Its internal fields, such as `positionArg`, are immutable definitions of command arguments, not mutable data stores. Each call to `execute` operates independently on the arguments and world state provided at that moment.
-   **Thread Safety:** The class itself contains no mutable state and is therefore inherently thread-safe. However, its `execute` method interacts directly with the game world (`World`, `WorldChunk`). **WARNING:** All world-mutating operations are fundamentally unsafe if not performed on the main server thread. The command system guarantees that `execute` is called on the correct thread, but developers extending this class must not attempt to access the passed-in `WorldChunk` or `PrefabSpawnerState` from other threads.

## API Surface
The primary contract is defined by the methods that the command system and subclasses interact with.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(N) | Overrides the base command. Implements the target resolution logic (argument parsing or raycasting). Throws GeneralCommandException on failure. Delegates to the abstract `execute` on success. Complexity is dominated by the raycast (`TargetUtil.getTargetBlock`). |
| execute(context, chunk, state) | void | Varies | **Abstract Method.** Subclasses must implement this method. It is the entry point for custom logic that operates on the validated PrefabSpawnerState. |

## Integration Patterns

### Standard Usage
A developer should extend this class to create a new command that targets a prefab spawner. The only responsibility of the subclass is to implement the abstract `execute` method.

```java
// Example of a concrete implementation
public class MySpawnerInfoCommand extends TargetPrefabSpawnerCommand {

    public MySpawnerInfoCommand() {
        super("spawnerinfo", "Gets information about a targeted prefab spawner.");
    }

    @Override
    protected void execute(@Nonnull CommandContext context, @Nonnull WorldChunk chunk, @Nonnull PrefabSpawnerState state) {
        // Custom logic here. The 'state' is guaranteed to be a valid PrefabSpawnerState.
        String prefabName = state.getPrefabName();
        context.sendMessage(Message.text("Spawner is configured for prefab: " + prefabName));
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new MySpawnerInfoCommand()` in your game logic. Commands are managed by the command system and should only be registered, not instantiated on-demand.
-   **Manual Execution:** Never call the `execute(context, world, store)` method directly. Bypassing the command system breaks permission checks, context setup, and critical thread safety guarantees.
-   **Incorrect State Assumption:** Do not attempt to cast a generic `BlockState` to `PrefabSpawnerState` yourself. This class's primary purpose is to perform that check safely for you before your logic is ever invoked.

## Data Pipeline
The flow of data for this command begins with user input and ends with a world interaction or a message back to the user.

> Flow:
> Player Chat Input -> Command Dispatcher -> **TargetPrefabSpawnerCommand.execute(context, world, store)** -> Argument Parser OR TargetUtil Raycast -> World State Query -> BlockState Validation -> **Subclass.execute(context, chunk, state)** -> CommandContext.sendMessage (Feedback)

