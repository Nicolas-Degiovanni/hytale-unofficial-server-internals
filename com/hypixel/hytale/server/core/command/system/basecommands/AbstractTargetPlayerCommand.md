---
description: Architectural reference for AbstractTargetPlayerCommand
---

# AbstractTargetPlayerCommand

**Package:** com.hypixel.hytale.server.core.command.system.basecommands
**Type:** Template / Base Class

## Definition
```java
// Signature
public abstract class AbstractTargetPlayerCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The AbstractTargetPlayerCommand is a foundational component within the server's command framework. It embodies the Template Method design pattern to provide a robust, reusable skeleton for any command that needs to operate on a player entity.

Its primary architectural role is to abstract away the complex and error-prone logic of target resolution, permission checking, and thread context switching. By extending this class, developers can focus solely on the business logic of their command, confident that the underlying execution context is safe and correct.

The core responsibilities of this class are:
1.  **Target Resolution:** It transparently handles identifying the target player. If a player argument is provided in the command, it is parsed and used. If omitted, the command defaults to targeting the entity that executed the command.
2.  **Permission Enforcement:** It automatically enforces the standard *.other* permission convention. If a command targets another player, the sender must have the elevated permission node (e.g., *hytale.command.kill.other*).
3.  **Context Validation:** It ensures the resolved target player is valid and currently exists within a world before proceeding.
4.  **Thread Safety:** Crucially, it marshals the final execution of the command logic onto the specific thread that owns the target player's world. This is the cornerstone of the server's concurrency model, preventing race conditions and ensuring safe mutation of world state.

Subclasses are prevented from overriding this core orchestration logic via the final `executeAsync` method, and are instead required to implement the abstract `execute` method, which is invoked by the template at the correct time and on the correct thread.

### Lifecycle & Ownership
-   **Creation:** This abstract class is never instantiated directly. Concrete implementations (e.g., a new `HealCommand`) are instantiated once by the server's Command Registry during the server bootstrap sequence.
-   **Scope:** Instances of concrete subclasses are effectively singletons managed by the Command Registry. They persist for the entire lifetime of the server.
-   **Destruction:** The command instances are garbage collected when the server shuts down and the Command Registry is cleared.

## Internal State & Concurrency
-   **State:** This class is stateless regarding command execution. It holds an immutable definition for the optional player argument (`playerArg`), but all runtime data is passed via the `CommandContext` parameter. This design ensures that a single command instance can safely process executions from many different players concurrently.
-   **Thread Safety:** The class is inherently thread-safe. The public-facing `executeAsync` method can be invoked from any network or main server thread. Its primary responsibility is to safely dispatch the actual work to the appropriate single-threaded `World` context via the parent `runAsync` scheduler. Implementers of the `execute` method can operate under the assumption that they are executing safely on the target's world thread.

## API Surface
The primary API surface is the contract that subclasses must fulfill.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, sourceRef, targetRef, playerRef, world, store) | void | Varies | **Abstract hook method.** Implement this to define the command's logic. Guaranteed to execute on the target player's world thread with a fully resolved and validated context. |

## Integration Patterns

### Standard Usage
A developer should extend this class and implement the `execute` method. The framework provides all necessary resolved objects.

```java
// Example: A simplified command to set a player's health.
public class SetHealthCommand extends AbstractTargetPlayerCommand {

    private final IntArg healthAmount = withRequiredArg("amount", "...", ArgTypes.INTEGER);

    public SetHealthCommand() {
        super("sethealth", "Sets a player's health.");
    }

    @Override
    protected void execute(
        @Nonnull CommandContext context,
        @Nullable Ref<EntityStore> sourceRef,
        @Nonnull Ref<EntityStore> targetRef,
        @Nonnull PlayerRef playerRef,
        @Nonnull World world,
        @Nonnull Store<EntityStore> store
    ) {
        // The framework guarantees targetRef is valid and we are on the correct world thread.
        int amount = this.healthAmount.get(context);
        
        // Safely apply logic to the target entity.
        // e.g., store.getComponent(targetRef, Health.class).setCurrent(amount);

        context.sendMessage(Message.translation("command.success"));
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Player Parsing:** Do not attempt to parse player names from the `CommandContext` inside the `execute` method. The framework has already performed this logic; rely on the provided `targetRef` and `playerRef` parameters.
-   **Ignoring Provided Context:** Do not attempt to re-fetch the `World` or `EntityStore` for the target. The framework provides these to ensure you are operating on the correct, thread-safe instances.
-   **Blocking Operations:** Do not perform long-running or blocking operations (e.g., disk I/O, complex database queries) directly within the `execute` method. This will stall the world thread for the target player, causing severe performance degradation. Offload such work to an asynchronous task.

## Data Pipeline
The flow of data and control through this component is strictly defined to ensure safety and correctness.

> Flow:
> Raw Command String -> Command Parser -> **AbstractTargetPlayerCommand.executeAsync** -> (Target Resolution & Permission Check) -> World Thread Scheduler -> **ConcreteCommand.execute** -> World State Mutation

