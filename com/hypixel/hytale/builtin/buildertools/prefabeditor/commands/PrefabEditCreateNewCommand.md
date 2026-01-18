---
description: Architectural reference for PrefabEditCreateNewCommand
---

# PrefabEditCreateNewCommand

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabEditCreateNewCommand extends AbstractAsyncPlayerCommand {
```

## Architecture & Concepts
The PrefabEditCreateNewCommand class implements the **Command** design pattern, serving as the primary user-facing entry point for initiating a new prefab editing session. It functions as a thin controller, responsible for translating a player's in-game chat command into a structured, asynchronous task.

Its core architectural role is to decouple the server's command parsing system from the complex business logic of the Prefab Editor. The command's responsibilities are strictly limited to:
1.  Defining the command's syntax, including required and optional arguments.
2.  Parsing and validating player-provided input from the CommandContext.
3.  Assembling a PrefabEditorCreationSettings Data Transfer Object (DTO).
4.  Delegating the core session creation logic to the central PrefabEditSessionManager.

This separation ensures that the command object remains simple and focused, while the session manager can handle the heavyweight operations of world modification, entity management, and player state changes without being coupled to the command system.

### Lifecycle & Ownership
-   **Creation:** A single instance of PrefabEditCreateNewCommand is instantiated by the server's Command System during plugin loading and registration. It is not created on a per-player or per-request basis.
-   **Scope:** The object is a long-lived singleton, persisting for the entire server session. It is registered with the command dispatcher and remains available to handle all incoming `/editprefab new` commands.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the BuilderToolsPlugin is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. Its member fields (RequiredArg, DefaultArg) are configuration objects that define the command's arguments. They are initialized once in the constructor and are treated as immutable for the lifetime of the object. All state relevant to a specific execution is passed into the executeAsync method via its parameters.

-   **Thread Safety:** The class is inherently thread-safe. By extending AbstractAsyncPlayerCommand, the framework guarantees that the executeAsync method is invoked on a dedicated worker thread, preventing any blockage of the main server tick loop. Since the object has no mutable instance state, multiple commands can be processed concurrently without risk of race conditions or data corruption.

## API Surface
The primary public contract is its registration as a command, not direct programmatic invocation. The executeAsync method is the entry point for the command system framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context, store, ref, playerRef, world) | CompletableFuture<Void> | O(1) | Framework-invoked method. Parses arguments and triggers the asynchronous creation of a new prefab editing session. The complexity of this method is constant; however, the downstream operation it initiates in the PrefabEditSessionManager is highly complex. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is invoked by a player with appropriate permissions executing the command in the game client. The framework handles routing the command to this class.

**Example In-Game Usage:**
```
/editprefab new MyNewDungeon asset
```
This command instructs the system to create a new, empty prefab session for a file named `MyNewDungeon.prefab.json` located in the standard asset directory.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new PrefabEditCreateNewCommand()`. The command system manages the lifecycle of registered commands. Direct instantiation creates an object that is not registered and will not function.
-   **Manual Invocation:** Do not call the `executeAsync` method directly. Doing so bypasses the entire command framework, including permission checks, argument parsing, and context provision, leading to unpredictable behavior and likely NullPointerExceptions.
-   **Stateful Modification:** Do not attempt to modify the argument definition fields (e.g., `prefabNameArg`) after construction. These are intended to be immutable configurations.

## Data Pipeline
The command acts as the initial stage in the prefab creation data pipeline. It translates unstructured player input into a structured request for the core builder tools system.

> Flow:
> Player Chat Input (`/editprefab new ...`) -> Server Command Parser -> **PrefabEditCreateNewCommand.executeAsync** -> PrefabEditorCreationSettings (DTO) -> PrefabEditSessionManager -> Asynchronous World Generation & Player Session Setup

