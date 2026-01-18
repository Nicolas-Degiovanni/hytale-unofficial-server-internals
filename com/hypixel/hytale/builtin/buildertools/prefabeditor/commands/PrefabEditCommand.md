---
description: Architectural reference for PrefabEditCommand
---

# PrefabEditCommand

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.commands
**Type:** Composite Command

## Definition
```java
// Signature
public class PrefabEditCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The PrefabEditCommand class serves as the primary entry point and namespace for all commands related to the in-game Prefab Editor. It does not implement any game logic itself. Instead, it functions as a **Composite** or a **dispatcher** in the command system.

Its sole responsibility is to aggregate a collection of specialized sub-commands (e.g., PrefabEditLoadCommand, PrefabEditSaveCommand) under a single, user-facing command: *editprefab*. When a player executes a command like `/editprefab load my_structure`, the server's command system first routes the request to this parent object. PrefabEditCommand then delegates the execution to the appropriate registered sub-command, in this case, an instance of PrefabEditLoadCommand.

This architectural pattern decouples the high-level command registration from the low-level implementation of each specific action. It allows for a clean, modular, and easily extensible command hierarchy for complex toolsets.

### Lifecycle & Ownership
-   **Creation:** Instantiated once by the server's command registration system during server startup or plugin initialization. It is discovered and registered automatically.
-   **Scope:** Session-scoped. The instance persists for the entire lifetime of the running server, held as a reference within the central command registry.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server shuts down or if the command is explicitly unregistered at runtime.

## Internal State & Concurrency
-   **State:** **Stateless and Immutable.** The configuration of this class, including its name, aliases, and the list of its sub-commands, is defined entirely within its constructor. This state does not change after instantiation.
-   **Thread Safety:** **Inherently Thread-Safe.** As a stateless object, PrefabEditCommand can be safely accessed by multiple threads without locks or synchronization.

    **WARNING:** While this container class is thread-safe, the sub-commands it dispatches to (e.g., PrefabEditSaveCommand) may not be. The overall thread safety of a prefab editing operation is determined by the implementation of the specific sub-command being executed.

## API Surface
This class exposes no unique public API. Its entire behavior is configured during construction and driven by the inherited contract from AbstractCommandCollection, which is in turn invoked by the server's core command processing system. Direct interaction with instances of this class is not a supported use case.

## Integration Patterns

### Standard Usage
This class is not designed to be used directly in code. It is a framework component that is activated by player input through the game's chat console. The standard interaction is performed by a server administrator or a player with appropriate permissions.

```text
// Player enters one of the following commands in the game client:
/editprefab load my_house
/pedit save
/prefabedit info
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new PrefabEditCommand()`. The server's command framework is solely responsible for the lifecycle of command objects. Manual instantiation will result in a non-functional, unregistered command.
-   **Extending for Logic:** Do not extend this class to add new logic. To add a new sub-command, create a new class that implements the required command interface and add it to the collection via the `addSubCommand` method within this class's constructor.

## Data Pipeline
The PrefabEditCommand acts as a routing point in the data flow from player input to world modification. It parses the initial command segment and delegates the remaining arguments to the appropriate handler.

> Flow:
> Player Chat Input -> Server Network Listener -> Command Parsing Engine -> **PrefabEditCommand (Dispatcher)** -> Sub-Command Handler (e.g., PrefabEditSaveCommand) -> Prefab System -> World State Mutation

