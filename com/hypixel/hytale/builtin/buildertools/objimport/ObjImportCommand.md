---
description: Architectural reference for ObjImportCommand
---

# ObjImportCommand

**Package:** com.hypixel.hytale.builtin.buildertools.objimport
**Type:** Transient

## Definition
```java
// Signature
public class ObjImportCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The ObjImportCommand class serves as a server-side entry point for initiating the Wavefront OBJ model import process. It is a concrete implementation within the server's command system, designed to be invoked directly by a player through the chat console.

Architecturally, this class is a *trigger* and a *gatekeeper*, not a worker. Its sole responsibility is to validate the command's context (e.g., player permissions, game mode) and then delegate the subsequent logic to the user interface layer. Upon successful execution, it instructs the player's session to open the ObjImportPage, a dedicated UI component that handles the file selection and import configuration. This decouples the command invocation from the complex, stateful UI interaction required for the import process.

## Lifecycle & Ownership
- **Creation:** A single instance of ObjImportCommand is instantiated by the server's command registration system during the server bootstrap phase. The system discovers command classes and registers them for runtime dispatch.
- **Scope:** The registered instance is a singleton that persists for the entire server lifecycle. The `execute` method, however, is invoked in a transient, per-request context each time a player runs the command.
- **Destruction:** The instance is dereferenced and garbage collected when the server shuts down or the module containing it is unloaded.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its internal fields (command name, aliases, permissions) are configured once in the constructor and are immutable thereafter. The `execute` method operates exclusively on the parameters provided by the command system, holding no state between invocations.
- **Thread Safety:** The object is inherently thread-safe due to its immutable and stateless nature. The Hytale server's command dispatcher ensures that the `execute` method is called from a safe thread context, typically the main server thread, which has exclusive access to the world state arguments like `Store` and `World`.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Overrides the framework method to handle command invocation. Retrieves the Player component and opens the ObjImportPage UI for them. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is invoked automatically by the server's command processing pipeline when a player types the command into chat.

*Player enters the following in the game client's chat console:*
```
/importobj
```
*or*
```
/obj
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ObjImportCommand()` in your own code. An instance created this way will not be registered with the server's command dispatcher and will have no effect.
- **Manual Invocation:** Avoid calling the `execute` method directly. Doing so bypasses the critical permission checks, context setup, and argument validation performed by the command system, leading to unpredictable behavior or server instability.

## Data Pipeline
ObjImportCommand acts as a control-flow initiator. It does not process data itself but instead triggers a UI-driven data flow.

> Flow:
> Player Chat Input -> Server Command Dispatcher -> **ObjImportCommand.execute()** -> Player.PageManager -> Network Message (Open UI) -> Client Renders ObjImportPage

