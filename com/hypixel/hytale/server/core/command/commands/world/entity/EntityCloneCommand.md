---
description: Architectural reference for EntityCloneCommand
---

# EntityCloneCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Transient

## Definition
```java
// Signature
public class EntityCloneCommand extends AbstractWorldCommand {
```

## Architecture & Concepts

The EntityCloneCommand is a server-side command responsible for duplicating entities within a world. It functions as a concrete implementation within the server's Command System, inheriting from AbstractWorldCommand to gain access to the necessary world and entity storage contexts.

Architecturally, this class serves two primary purposes:
1.  **Command Definition:** It declares the command's public interface for users and the system, including its name (*clone*), description, and arguments. It uses the argument builder pattern (e.g., withOptionalArg, withDefaultArg) to define typed parameters like the target entity and the number of copies, complete with validators.
2.  **Execution Logic:** It contains the business logic to safely and correctly duplicate an entity. This is not a simple memory copy. The process involves a deep copy of the target entity's component data via the EntityStore, followed by a **critical** re-initialization of its unique identifier (UUIDComponent). This prevents entity identity collisions, which would corrupt world state.

The class exposes its core duplication logic via a public static utility method, cloneEntity, allowing other server systems to perform entity cloning programmatically without invoking the full command parsing and execution pipeline.

### Lifecycle & Ownership
-   **Creation:** A single instance of EntityCloneCommand is instantiated by the Command System during server bootstrap. The system discovers and registers all classes that extend a command base class like AbstractWorldCommand.
-   **Scope:** The object instance is a long-lived singleton that persists for the entire server session. However, its role is that of a transient handler; it holds no mutable state between invocations. All state is passed into the execute method for the duration of a single command execution.
-   **Destruction:** The instance is discarded during server shutdown. No explicit cleanup logic is required.

## Internal State & Concurrency
-   **State:** The class instance is effectively immutable after construction. Its fields, such as entityArg and countArg, hold the configuration for command argument parsing and are not modified after the constructor runs. The execute method is stateless and idempotent, operating exclusively on the arguments provided to it.
-   **Thread Safety:** This class is thread-safe under the assumption that the server's Command System invokes the execute method from a single, main game thread. Direct, concurrent invocations of the execute method on a shared instance would be unsafe, as the underlying World and Store objects it manipulates are not guaranteed to be safe for concurrent modification from arbitrary threads. The static cloneEntity method carries the same thread-safety constraints as the underlying EntityStore it modifies.

## API Surface
The primary contract is with the command framework via the protected execute method. A secondary public contract is offered via the static utility method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | protected void | O(N \* C) | Framework entry point. Executes the command logic. N is the clone count, C is the entity component count. |
| cloneEntity(sender, ref, store) | public static void | O(C) | Utility function to perform a single entity clone. C is the entity component count. |

## Integration Patterns

### Standard Usage
The primary integration is through the server's command line interface. For programmatic use by other systems (e.g., a scripted event or spawner logic), the static cloneEntity method is the correct entry point.

```java
// Programmatic cloning via the static utility method
// Assumes 'world' and 'targetEntityRef' are already acquired

Store<EntityStore> entityStore = world.get(EntityStore.class);

// This is the correct way to clone an entity in server-side code
EntityCloneCommand.cloneEntity(serverSender, targetEntityRef, entityStore);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new EntityCloneCommand()`. The instance is managed by the Command System. Creating a new one has no effect and will not register a command.
-   **Manual Invocation of Execute:** Do not call the `execute` method directly. This bypasses critical infrastructure such as permission checks, argument parsing, and context validation provided by the command dispatcher.
-   **Manual Cloning Logic:** Avoid replicating the cloning logic found inside this command. Forgetting to replace the UUIDComponent after a copy will lead to severe and difficult-to-diagnose bugs related to entity identity conflicts. Always use the provided `cloneEntity` static method.

## Data Pipeline
The flow for a user-initiated command is a clear, sequential process managed by the Command System.

> Flow:
> Player Chat Input (`/clone @p 5`) -> Network Packet -> Command Dispatcher -> **EntityCloneCommand** (Argument Parsing) -> **EntityCloneCommand.execute** (Logic) -> EntityStore API (copyEntity, addEntity) -> World State Change

