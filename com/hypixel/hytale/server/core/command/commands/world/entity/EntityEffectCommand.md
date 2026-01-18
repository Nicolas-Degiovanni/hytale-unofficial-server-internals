---
description: Architectural reference for EntityEffectCommand
---

# EntityEffectCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Command Handler

## Definition
```java
// Signature
public class EntityEffectCommand extends AbstractTargetEntityCommand {
```

## Architecture & Concepts
The EntityEffectCommand is a concrete implementation within the server's Command System. It serves as a direct interface between user-level text commands and the underlying Entity Component System (ECS).

Architecturally, this class follows the Command Pattern. It encapsulates a single server action—applying a status effect to an entity—into a standalone object. Its inheritance from AbstractTargetEntityCommand is a critical design choice, abstracting away the complex logic of parsing entity selectors (e.g., `@p`, `@a`, or specific entity names) and resolving them into a list of target entities.

The primary responsibility of this class is to:
1.  Define the required and optional arguments for the `/effect` command (the effect type and duration).
2.  Translate these parsed arguments from the CommandContext into concrete game data types (EntityEffect and float).
3.  Iterate over the resolved target entities and interact with their EffectControllerComponent to apply the desired effect.

This class acts as a clean translation layer, preventing the core ECS from needing any knowledge of the command system's implementation details.

### Lifecycle & Ownership
- **Creation:** A single instance of EntityEffectCommand is created by the server's command registration service during the server bootstrap sequence. It is not created on-demand per command execution.
- **Scope:** The instance is a long-lived object that persists for the entire server session. Its state, which consists of its argument definitions, is configured once at creation and is immutable thereafter.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server is shutting down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** This class contains state in the form of its argument definitions (effectArg, durationArg). However, this state is configured within the constructor and is effectively immutable for the object's lifetime. The execute method itself is stateless and operates exclusively on the parameters passed to it, ensuring that each command execution is independent.
- **Thread Safety:** This class is **not thread-safe** and is designed to be executed exclusively on the main server thread, which manages the world tick. The underlying ECS components, such as EffectControllerComponent, are not designed for concurrent access. Invoking the execute method from an asynchronous thread will lead to race conditions, data corruption, and server instability.

## API Surface
The primary contract is the protected `execute` method, invoked by the parent AbstractTargetEntityCommand after it has resolved the target entities.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, entities, world, store) | void | O(N) | Applies the configured effect to N target entities. This method is the core logic entry point. It will fail silently for entities that lack an EffectControllerComponent. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by developers. It is automatically discovered and invoked by the server's command handling system in response to a player or console input.

The standard interaction is through the in-game chat or server console:
```
/effect @a hytale:poison 30
```
This input is parsed by the command system, which then identifies EntityEffectCommand as the handler, populates a CommandContext with the parsed arguments, resolves the `@a` selector into a list of all players, and finally invokes the `execute` method on the singleton instance of this class.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new EntityEffectCommand()`. The command system manages the lifecycle of all command handlers. Manually creating an instance will result in an object that is not registered with the server and will never be used.
- **Manual Execution:** Avoid calling the `execute` method directly. Doing so bypasses the framework's essential setup, including argument parsing, entity selection, and permission validation. This will almost certainly result in a NullPointerException or other undefined behavior as the CommandContext and entity list will not be properly populated.

## Data Pipeline
The flow of data for a typical `/effect` command execution is linear and synchronous, operating entirely within a single server tick.

> Flow:
> Player Input (`/effect ...`) -> Network Packet -> Server Command Parser -> **EntityEffectCommand** -> EffectControllerComponent.addEffect -> Entity State Modification

