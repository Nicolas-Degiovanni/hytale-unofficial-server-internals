---
description: Architectural reference for EntityIntangibleCommand
---

# EntityIntangibleCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Singleton

## Definition
```java
// Signature
public class EntityIntangibleCommand extends AbstractTargetEntityCommand {
```

## Architecture & Concepts
The EntityIntangibleCommand is a server-side command handler responsible for manipulating the **Intangible** component on game entities. It serves as a high-level abstraction layer, translating a text-based command from a user or script into a direct modification of the server's Entity-Component-System (ECS) state.

This class extends AbstractTargetEntityCommand, inheriting a standardized framework for parsing entity selectors (like @e for all entities) and applying an operation to the resulting list. Its sole function is to add or remove the Intangible component, which typically governs whether an entity can be collided with or passed through. This command is a critical tool for server administrators and game designers to dynamically alter entity physics properties at runtime.

## Lifecycle & Ownership
- **Creation:** A single instance of EntityIntangibleCommand is created by the server's command registration system during the server bootstrap sequence. It is discovered and registered to handle the `intangible` subcommand within the entity command namespace.
- **Scope:** The object is a stateless singleton that persists for the entire server session. Its lifetime is tied directly to the command registry itself.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless from an execution perspective. It holds a final field, removeFlag, which is an immutable definition of a command-line argument (`--remove`). This state is configured once at construction and never changes. All state required for execution (the target entities, the world, the component store) is passed into the `execute` method.
- **Thread Safety:** **This class is not thread-safe.** The `execute` method performs direct write operations on the World's EntityStore. Such operations must be exclusively performed on the main server thread to prevent catastrophic race conditions with physics, AI, and other systems that read component data every tick. The command system guarantees that all command execution is synchronized with the main game loop.

## API Surface
The public contract is defined by the parent class. The core logic is implemented in the protected `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, entities, world, store) | void | O(N) | Modifies the Intangible component for a list of N entities. Adds the component by default, or removes it if the `--remove` flag is present. Sends a success message to the command context. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly. It is automatically executed by the server's command processing system in response to in-game commands. A server administrator or a script would trigger its execution.

**Example Command:**
```plaintext
# Makes the nearest player intangible
/entity intangible @p

# Makes all zombies tangible again
/entity intangible @e[type=zombie] --remove
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EntityIntangibleCommand()`. The command system manages its lifecycle. Creating a new instance serves no purpose as it will not be registered to handle any command strings.
- **Manual Invocation:** Calling the `execute` method directly from other game systems is a severe anti-pattern. This bypasses critical infrastructure for permission checking, argument parsing, and target selection, potentially corrupting game state or creating security holes. Use the dedicated ECS APIs for direct component manipulation.

## Data Pipeline
The flow of data for this command is unidirectional, originating from an external command source and resulting in a world state change.

> Flow:
> Player Chat or Console Input -> Command System Parser -> Target Selector Resolution -> **EntityIntangibleCommand.execute()** -> EntityStore Component Write -> World State Update

