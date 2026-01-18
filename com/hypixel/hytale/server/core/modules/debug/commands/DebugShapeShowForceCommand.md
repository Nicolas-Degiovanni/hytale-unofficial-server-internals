---
description: Architectural reference for DebugShapeShowForceCommand
---

# DebugShapeShowForceCommand

**Package:** com.hypixel.hytale.server.core.modules.debug.commands
**Type:** Transient

## Definition
```java
// Signature
public class DebugShapeShowForceCommand extends CommandBase {
```

## Architecture & Concepts
The DebugShapeShowForceCommand is a concrete implementation of the **Command Pattern**. It encapsulates a single, specific server-side action—toggling the rendering of debug force vectors for physics shapes—into a self-contained object.

This class is designed to be discovered and managed by the server's central **CommandSystem**. It acts as a terminal node in the command processing pipeline, translating a user-facing text command into a direct modification of global debug state. Its primary architectural role is to decouple the command invocation source (e.g., a player's chat input) from the system being manipulated (the DebugUtils state).

By extending CommandBase, it adheres to a strict contract for registration and execution, allowing the CommandSystem to handle its lifecycle, argument parsing, and thread scheduling transparently.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the server's CommandSystem during the bootstrap phase. The system likely uses reflection or a service locator to find all subclasses of CommandBase and register them.
- **Scope:** The object's lifecycle is tied to the server session. It is created on startup and persists until shutdown, effectively acting as a singleton managed by the command registry.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the CommandSystem is torn down during server shutdown.

## Internal State & Concurrency
- **State:** This class is **stateless**. It maintains no internal fields and its behavior does not change based on previous executions. Its sole purpose is to modify external, static state located in the DebugUtils class.

- **Thread Safety:** The `executeSync` method is, by framework contract, guaranteed to be invoked exclusively on the main server thread. This obviates the need for internal locking mechanisms within this class.
    - **WARNING:** The state it modifies, `DebugUtils.DISPLAY_FORCES`, is a global static variable. If other systems, such as a separate physics or rendering thread, read this value, they must be prepared to handle concurrency. The `DISPLAY_FORCES` flag itself should be declared as volatile or be protected by locks within the DebugUtils class to ensure visibility across threads. Failure to do so can result in stale data and inconsistent debug rendering.

## API Surface
The public API is minimal and intended for framework use only. Direct invocation by application code is a design violation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(CommandContext) | protected void | O(1) | Toggles the global `DebugUtils.DISPLAY_FORCES` flag and sends a confirmation message to the command source. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is invoked automatically by the CommandSystem when a privileged user executes the corresponding command from a client or server console.

```
// User input in console
/showforce
```

The server's command dispatcher routes this input to the registered instance of DebugShapeShowForceCommand, which then executes its logic.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new DebugShapeShowForceCommand()`. The CommandSystem is the sole owner and manager of command objects. Direct instantiation will result in an un-registered command that is never executed.
- **Manual Execution:** Do not obtain an instance from the CommandSystem to call `executeSync` manually. This bypasses the framework's critical pre-processing steps, including permission checks, argument validation, and thread synchronization.

## Data Pipeline
The flow of data for this command is linear, originating from user input and resulting in a global state change and a feedback message.

> Flow:
> User Input (`/showforce`) -> Network Listener -> Command Parser -> CommandSystem Dispatcher -> **DebugShapeShowForceCommand.executeSync** -> Global State Mutation (`DebugUtils.DISPLAY_FORCES`) -> Message Creation -> CommandContext -> Network Emitter -> Client Console

