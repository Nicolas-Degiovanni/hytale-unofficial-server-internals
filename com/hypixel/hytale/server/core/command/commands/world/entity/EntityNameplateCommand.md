---
description: Architectural reference for EntityNameplateCommand
---

# EntityNameplateCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Transient Processor

## Definition
```java
// Signature
public class EntityNameplateCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The EntityNameplateCommand is a server-side command processor responsible for modifying the visible nameplate of an in-world entity. It serves as a direct interface between a user-invoked command and the server's core Entity Component System (ECS).

As a subclass of AbstractWorldCommand, it operates within the server's command dispatching framework. Its primary architectural role is to translate parsed user input into a specific, state-changing operation on the world's EntityStore. This class encapsulates the logic for identifying a target entity, accessing its component data store, and either creating, updating, or (via its inner class) removing a Nameplate component.

This command is not a persistent service; rather, it is a stateless handler whose instance is registered with the command system at startup. Each execution is an atomic operation within a single server tick.

## Lifecycle & Ownership
- **Creation:** A single instance of EntityNameplateCommand is created by the server's command registration system during the server bootstrap sequence. It is not instantiated on a per-request basis.
- **Scope:** The instance persists for the entire lifecycle of the running server. It is held in memory by the central command registry.
- **Destruction:** The object is marked for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** The class instance holds immutable state related to its argument definitions (entityArg, textArg). These are configured once in the constructor and never change. The execute method itself is stateless; it does not modify its own fields and all state it operates on is passed in via the CommandContext and World parameters.
- **Thread Safety:** This class is **not thread-safe** and must only be invoked by the main server thread. The execute method performs direct mutations on the EntityStore, which is a critical data structure managed by the server's primary game loop. Unsynchronized access from other threads would lead to world state corruption, race conditions, and server instability. The command system's design guarantees single-threaded execution.

## API Surface
The primary contract is the protected execute method, invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EntityNameplateCommand() | constructor | O(1) | Initializes the command, its name, description, and argument parsers. Registers the Remove subcommand as a usage variant. |
| execute(context, world, store) | void | O(log N) | Executes the command logic. Complexity is dominated by entity lookup in the EntityStore. Modifies the target entity's Nameplate component. |
| Remove (inner class) | class | N/A | A subcommand handler for removing an entity's nameplate. Follows the same architectural pattern. |

## Integration Patterns

### Standard Usage
This command is designed to be invoked by the server's command dispatcher in response to user input. A developer should never call its methods directly.

```java
// This is a conceptual example of how the system uses the command.
// A developer would not write this code.
// User input: /nameplate @p "Hello World"

// 1. CommandDispatcher parses input and finds the EntityNameplateCommand instance.
// 2. It populates a CommandContext with the parsed arguments.
// 3. It invokes the execute method on the main server thread.
command.execute(populatedContext, currentWorld, worldEntityStore);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EntityNameplateCommand()`. The server's command registry manages the lifecycle of all command objects. Direct instantiation will result in a non-functional command that is not registered with the dispatcher.
- **Manual Execution:** Never call the `execute` method directly. Doing so bypasses critical infrastructure, including permission checks, context setup, and thread safety guarantees provided by the command dispatcher. This can lead to unpredictable behavior and server crashes.
- **Stateful Logic:** Do not modify this class to hold state across multiple executions. Command handlers must be stateless to ensure predictable behavior.

## Data Pipeline
The flow of data for this command is unidirectional, from user input to world state modification.

> Flow:
> User Input (`/nameplate ...`) -> Network Layer -> Server Command Parser -> **EntityNameplateCommand.execute()** -> EntityStore Component Mutation -> (Potentially) Network Packet to update client view

---
description: Architectural reference for EntityNameplateCommand.Remove
---

# EntityNameplateCommand.Remove

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Transient Processor

## Definition
```java
// Signature
public static class Remove extends AbstractWorldCommand {
```

## Architecture & Concepts
The Remove class is a static nested class of EntityNameplateCommand that functions as a dedicated subcommand. It is registered as a "usage variant" of the parent command, typically invoked via a syntax like `/nameplate remove <entity>`.

Architecturally, it mirrors its parent: it is a stateless, server-side command processor that translates user input into a specific ECS operation. Its sole responsibility is to remove the Nameplate component from a target entity, effectively hiding its nameplate. By encapsulating this logic separately, the system maintains a clean separation of concerns between the "update" and "remove" actions while keeping them logically grouped under the `nameplate` command.

## Lifecycle & Ownership
- **Creation:** A single instance is created by the parent EntityNameplateCommand constructor and passed to the `addUsageVariant` method.
- **Scope:** Its lifecycle is directly tied to its parent instance. It persists for the entire server session, held by the parent command's variant list within the command registry.
- **Destruction:** It is garbage collected when its parent EntityNameplateCommand instance is destroyed during server shutdown.

## Internal State & Concurrency
- **State:** The instance holds an immutable definition for its single optional argument, entityArg. The execute method is stateless.
- **Thread Safety:** Like its parent, this class is **not thread-safe**. It must only be invoked on the main server thread to prevent corruption of the EntityStore.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Remove() | constructor | O(1) | Initializes the subcommand's description and argument parser. |
| execute(context, world, store) | void | O(log N) | Executes the removal logic. Locates the target entity and calls `tryRemoveComponent` for the Nameplate component type. |

## Integration Patterns

### Standard Usage
This subcommand is invoked transparently by the command dispatcher when the user input matches its registered pattern.

```java
// Conceptual flow for user input: /nameplate remove @p

// 1. CommandDispatcher identifies the base "nameplate" command.
// 2. It then checks registered variants and matches the "remove" keyword.
// 3. The dispatcher invokes the execute method on the Remove instance.
removeSubcommand.execute(populatedContext, currentWorld, worldEntityStore);
```

### Anti-Patterns (Do NOT do this)
- **Direct Usage:** All anti-patterns applicable to the parent EntityNameplateCommand also apply here. Never instantiate or execute this class manually. It is an internal component of the command system.

## Data Pipeline
The data flow is identical to its parent, but the final operation is a component removal instead of an update.

> Flow:
> User Input (`/nameplate remove ...`) -> Network Layer -> Server Command Parser -> **EntityNameplateCommand.Remove.execute()** -> EntityStore Component Removal -> (Potentially) Network Packet to update client view

