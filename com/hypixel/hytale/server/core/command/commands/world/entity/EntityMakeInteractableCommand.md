---
description: Architectural reference for EntityMakeInteractableCommand
---

# EntityMakeInteractableCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Singleton

## Definition
```java
// Signature
public class EntityMakeInteractableCommand extends AbstractTargetEntityCommand {
```

## Architecture & Concepts
The EntityMakeInteractableCommand is a concrete implementation of the Command design pattern, specifically tailored for server-side world modification. It exists within the server's command processing framework and is responsible for a single, atomic action: toggling the `Interactable` component on game entities.

Architecturally, this class demonstrates the principle of **inheritance-based specialization**. It extends AbstractTargetEntityCommand, which provides the complex and reusable logic for parsing entity selectors (e.g., `@e[type=player]`, `@p`) and resolving them into a list of concrete entity references. EntityMakeInteractableCommand inherits this targeting capability and only implements the final action to be performed on the resolved entities.

This design cleanly separates the "what" (making an entity interactable) from the "who" (which entities to target). The command acts as a direct bridge between a user-initiated command and the server's core Entity Component System (ECS). By adding or removing the Interactable component, it flags an entity for other game systems, such as the player interaction handler, to recognize and act upon.

### Lifecycle & Ownership
- **Creation:** A single instance of EntityMakeInteractableCommand is instantiated by the server's CommandRegistry during the server bootstrap phase. The registry scans for command classes and creates long-lived instances to handle all subsequent invocations.
- **Scope:** The object's lifecycle is tied to the server session. It persists as long as the server is running.
- **Destruction:** The instance is de-referenced and eligible for garbage collection only when the server shuts down and the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless regarding its execution. Its only member field, `disableFlag`, is an immutable configuration object initialized in the constructor. All state required for an operation (the command context, the target entities, the world reference) is passed as arguments to the `execute` method. This design ensures that a single instance can safely handle multiple, sequential command executions without retaining state between them.
- **Thread Safety:** This class is **not thread-safe**. It performs direct mutations on the world state via the `Store<EntityStore>` object. All ECS modifications are fundamentally unsafe if not performed on the main server thread. The command framework is responsible for ensuring that the `execute` method is always invoked from the correct, synchronized game loop thread.

**WARNING:** Never invoke the `execute` method from an asynchronous task or a different thread. Doing so will lead to world corruption, race conditions, and server instability.

## API Surface
The primary contract is the `execute` method, which is invoked by the command framework after argument parsing and entity selection are complete.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, entities, world, store) | protected void | O(N) | Iterates through N target entities, adding or removing the Interactable component. Sends a feedback message to the context. |

## Integration Patterns

### Standard Usage
Developers do not typically instantiate or invoke this class directly. It is designed to be discovered and managed by the command system. The standard integration is to ensure the class is available for the system's classpath scanning during server startup.

A user or system triggers the command via the server console or in-game chat:
```
# Makes the nearest entity interactable
/entity interactable @e[limit=1,sort=nearest]

# Disables interaction on all pigs
/entity interactable @e[type=pig] --disable
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EntityMakeInteractableCommand()`. The command system manages the lifecycle of command objects. Manual instantiation serves no purpose and will not be registered to handle commands.
- **Manual Execution:** Never call the `execute` method directly. Bypassing the command framework skips critical steps like permission checking, entity selector parsing, and argument validation. This can result in operating on incorrect entities or producing an inconsistent world state.

## Data Pipeline
The flow of data from user input to world change is managed entirely by the server's command framework.

> Flow:
> User Command String (`/entity...`) -> Network Packet -> Server Command Parser -> CommandRegistry (resolves to **EntityMakeInteractableCommand**) -> AbstractTargetEntityCommand (parses selectors) -> **EntityMakeInteractableCommand.execute()** -> ECS Store Mutation -> Message System (sends feedback) -> Client UI Update

