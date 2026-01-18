---
description: Architectural reference for EntityHideFromAdventurePlayersCommand
---

# EntityHideFromAdventurePlayersCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Transient

## Definition
```java
// Signature
public class EntityHideFromAdventurePlayersCommand extends AbstractTargetEntityCommand {
```

## Architecture & Concepts
The EntityHideFromAdventurePlayersCommand is a concrete implementation within the server's Command System framework. It follows the Command Pattern, encapsulating a specific server action—modifying an entity's visibility state—into a standalone object.

Its primary architectural function is to serve as a bridge between user input (a chat command) and the server's Entity Component System (ECS). The class inherits from AbstractTargetEntityCommand, which is a critical design choice. This parent class abstracts away the complex and error-prone logic of parsing entity selectors (e.g., `@e[type=zombie]`) and resolving them into a concrete list of entity references. By doing so, this command can focus exclusively on its core responsibility: applying or removing the HiddenFromAdventurePlayers component.

This class is purely a state manipulator. It does not contain the logic for *what it means* for an entity to be hidden. That behavior is decoupled and handled by other core engine systems, such as the network replication or client visibility systems, which query for the presence of the HiddenFromAdventurePlayers component.

### Lifecycle & Ownership
- **Creation:** A single instance of this command is instantiated by the server's command registration system during the server bootstrap phase. It is discovered via reflection or an explicit registration call and mapped to the command string "hidefromadventureplayers".
- **Scope:** Application-scoped. The registered instance persists for the entire server session. It is designed to be stateless with respect to execution, allowing a single instance to be reused for all invocations of the command.
- **Destruction:** The instance is de-referenced and eligible for garbage collection when the server shuts down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** The class instance holds a final reference to a FlagArg object, which defines the "remove" flag. This state is configured during construction and is immutable thereafter. The command object itself does not store any per-execution state; all necessary data is passed as arguments to the execute method.
- **Thread Safety:** This class is inherently thread-safe. Its internal state is immutable post-construction. The execute method operates exclusively on parameters passed into its stack frame.

    **Warning:** While the command object is thread-safe, it operates on the shared ECS Store. The atomicity and thread safety of the underlying `store.ensureComponent` and `store.tryRemoveComponent` operations are critical for system stability. The command assumes and relies on the ECS Store to manage its own internal locking and concurrency control.

## API Surface
The primary contract is the protected `execute` method, which is invoked by the command framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, entities, world, store) | void | O(N) | Modifies the component state for a list of N target entities. Adds or removes the HiddenFromAdventurePlayers component based on the parsed "remove" flag. Sends a success message to the command context upon completion. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by game logic or plugin developers. It is automatically discovered and executed by the server's command processing pipeline in response to player or console input.

The framework is responsible for:
1. Parsing the raw command string.
2. Identifying this command as the target.
3. Resolving the entity selector argument into a list of entities.
4. Invoking the `execute` method with the fully prepared context.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EntityHideFromAdventurePlayersCommand()`. The command system manages the lifecycle of command objects. Attempting to create your own instance will result in an object that is not registered with the server and cannot be executed.
- **Manual Execution:** Avoid calling the `execute` method directly. Doing so bypasses the entire command framework, including critical steps like permission checking, argument validation, and entity selector resolution. This will lead to unpredictable behavior and likely throw NullPointerExceptions or IllegalStateExceptions.

## Data Pipeline
This command acts as a controller in a larger data flow, translating a user intention into a change in the world state.

> Flow:
> User Command String -> Network Packet -> Server Command Parser -> **EntityHideFromAdventurePlayersCommand** -> ECS Store Modification -> Visibility/Replication System Query

1. A user or system issues a command string like `/entity @a hidefromadventureplayers`.
2. The server's command parser dispatches the request to the registered EntityHideFromAdventurePlayersCommand instance.
3. The AbstractTargetEntityCommand superclass intercepts the call, parses the `@a` selector, and queries the World to produce a list of target entities.
4. The framework invokes this class's `execute` method with the resolved entity list.
5. The command iterates the list, calling `store.ensureComponent` on the central ECS Store for each entity.
6. Later, during the game tick, other systems (e.g., the Network Replication System) query the ECS Store. When processing entities for a client in Adventure Mode, they will detect the presence of the HiddenFromAdventurePlayers component and skip sending data for those entities to that client.

