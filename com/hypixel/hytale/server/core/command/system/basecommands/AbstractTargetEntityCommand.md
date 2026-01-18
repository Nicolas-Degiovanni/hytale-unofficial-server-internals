---
description: Architectural reference for AbstractTargetEntityCommand
---

# AbstractTargetEntityCommand

**Package:** com.hypixel.hytale.server.core.command.system.basecommands
**Type:** Framework Component (Template)

## Definition
```java
// Signature
public abstract class AbstractTargetEntityCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts

The AbstractTargetEntityCommand is a foundational base class within the server's command system. It embodies the **Template Method Pattern** to provide a robust, reusable framework for any command that needs to operate on one or more entities.

Its primary architectural purpose is to **decouple command logic from target resolution logic**. Subclasses are responsible only for implementing the specific action to be performed on the entities (e.g., teleport, modify data, remove). This base class handles the complex and error-prone process of identifying which entities the command should affect based on a variety of common selection criteria:

*   A specific entity ID.
*   A specific player.
*   All entities within a given radius of the command sender.
*   The entity currently in the command sender's line-of-sight.

By extending AbstractAsyncCommand, it ensures that all entity selection and subsequent operations are performed asynchronously and on the correct world thread, preventing server stalls and ensuring data integrity. It is a critical component for maintaining concurrency safety within the command system.

### Lifecycle & Ownership
- **Creation:** Concrete subclasses are instantiated once by the server's Command Registry during the bootstrap phase. They are not created on a per-command basis.
- **Scope:** The command object itself is a singleton that persists for the entire server session. The execution context, including resolved arguments and entity lists, is transient and scoped only to a single invocation of the command.
- **Destruction:** The object is eligible for garbage collection only when the server is shutting down and the Command Registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively **stateless and immutable** after its initial construction. It holds final fields defining the command's argument structure, but it does not store any mutable state related to a specific command execution. All execution-specific data is passed through the CommandContext or as local variables within the `executeAsync` method.

- **Thread Safety:** This class is **thread-safe by design** and acts as a critical concurrency control primitive.
    - The entry point, `executeAsync`, is designed to be called from any thread (typically a network or console thread).
    - It safely resolves the target World and then uses the `runAsync` scheduler to marshal the core logic, including the final call to the subclass `execute` method, onto the dedicated thread for that World.
    - **WARNING:** This guarantee of thread safety only applies if subclasses confine all world-mutating operations to within their implementation of the `execute` method. Accessing or modifying game state outside of this method from a command context is a severe anti-pattern that will lead to race conditions.

## API Surface

The primary API surface is not for external callers, but for subclasses that extend this class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AbstractTargetEntityCommand(name, desc, ...) | constructor | O(1) | Initializes the command with its name and description. Must be called by subclass constructors. |
| execute(context, entities, world, store) | protected abstract void | Varies | The core extension point. Subclasses **must** implement this method to define the action performed on the resolved list of entities. Guaranteed to be executed on the correct world thread. |

## Integration Patterns

### Standard Usage

A developer creates a new command by extending this class and implementing the `execute` method. The framework handles all argument parsing, target resolution, and thread scheduling.

```java
// Example: A command to set all targeted entities on fire.
public class SetFireCommand extends AbstractTargetEntityCommand {

    public SetFireCommand() {
        super("setfire", "Sets the targeted entity or entities on fire.");
    }

    @Override
    protected void execute(
        @Nonnull CommandContext context,
        @Nonnull ObjectList<Ref<EntityStore>> entities,
        @Nonnull World world,
        @Nonnull Store<EntityStore> store
    ) {
        for (Ref<EntityStore> entityRef : entities) {
            // Safely apply a FireComponent or similar logic here.
            // This code is guaranteed to be on the correct world thread.
        }
        context.sendMessage(Message.translation("command.success.setfire").param("count", entities.size()));
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Overriding executeAsync:** Do not attempt to override `executeAsync`. It is marked as `final` to enforce the class's control over the execution flow and thread safety. Bypassing it will break concurrency guarantees.
- **Manual Thread Management:** Do not introduce your own threading or futures within the `execute` method. The framework has already placed you on the correct thread.
- **Accessing World State Before `execute`:** Do not attempt to read or write component data or other world state before the `execute` method is called. The data is not guaranteed to be in a consistent state and you will not be on the correct thread.

## Data Pipeline

This class defines a clear, sequential data pipeline for processing an entity-targeting command.

> Flow:
> Player Chat Input -> Server Command Parser -> **AbstractTargetEntityCommand.executeAsync** (Target Resolution Logic) -> World Thread Scheduler -> **Subclass.execute** (Game Logic Implementation) -> Entity Component Store Mutation

