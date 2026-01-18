---
description: Architectural reference for NPCAppearanceCommand
---

# NPCAppearanceCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Handler

## Definition
```java
// Signature
public class NPCAppearanceCommand extends NPCWorldCommandBase {
```

## Architecture & Concepts
The NPCAppearanceCommand is a server-side command handler responsible for modifying the visual model of a target Non-Player Character entity. It serves as a specific, concrete implementation within the server's broader Command System framework.

This class acts as the terminal node in the command processing chain for the *appearance* subcommand. Its primary architectural role is to translate a structured command input, validated and parsed by the system, into a direct state change on a game world object, specifically an NPCEntity.

By inheriting from NPCWorldCommandBase, it delegates the complex and error-prone logic of identifying the target NPC and acquiring a transactional handle to the world's EntityStore. This allows NPCAppearanceCommand to focus exclusively on its core responsibility: parsing the model asset argument and invoking the corresponding setter on the entity. This follows the Single Responsibility Principle and promotes a clean separation of concerns between command routing, entity selection, and state mutation.

## Lifecycle & Ownership
- **Creation:** A single instance of NPCAppearanceCommand is instantiated by the server's central CommandRegistry during the server bootstrap and initialization phase. It is registered under the "appearance" subcommand within the "npc" command tree.

- **Scope:** The object instance is a long-lived singleton for the duration of the server's runtime. It is stateless and designed to be reused for every invocation of the corresponding command.

- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only during server shutdown, when the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless and immutable after construction. Its only member field, modelArg, is an argument *definition* object that is initialized once in the constructor and is not modified during runtime. All state modification occurs on external objects passed into the execute method, namely the NPCEntity.

- **Thread Safety:** This class is not thread-safe and is not designed to be. Command execution within the Hytale server is strictly serialized on a per-world basis, typically occurring on the main world thread during the game tick. The execute method must only be called from this thread. Any attempt to invoke it from an asynchronous task or worker thread will lead to race conditions, data corruption in the EntityStore, and world instability.

## API Surface
The public contract is defined by the overridden method from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, npc, world, store, ref) | void | O(1) | Executes the appearance change. Parses the required ModelAsset from the CommandContext and applies it to the target NPCEntity via its setAppearance method. This operation is transactional within the provided EntityStore context. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly via Java code. It is exclusively triggered by the server's command dispatcher when a user or system executes the corresponding command.

The standard interaction is through the server console or an in-game chat prompt.

```plaintext
# Selects an NPC with ID 123 and changes its model to the standard Kweebec model.
/npc select 123 appearance model=hytale:kweebec_male
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new NPCAppearanceCommand()`. The command system requires a single, registered instance to manage permissions, argument definitions, and command tree routing. Creating a new instance bypasses this entire framework.

- **Manual Execution:** Never call the `execute` method directly. Doing so circumvents the CommandContext setup, argument parsing, and transactional safety provided by the parent NPCWorldCommandBase and the wider command system. This will result in NullPointerExceptions and invalid world state.

## Data Pipeline
The flow of data for a typical appearance change command is linear and unidirectional, originating from user input and terminating in a replicated state change.

> Flow:
> Console Input (`/npc ...`) -> Command Dispatcher -> Argument Parser & Validator -> **NPCAppearanceCommand.execute()** -> NPCEntity.setAppearance() -> EntityStore State Mutation -> Network Replication System -> Client-Side Entity Update

