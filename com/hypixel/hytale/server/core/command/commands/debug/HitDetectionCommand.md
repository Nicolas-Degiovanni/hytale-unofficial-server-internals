---
description: Architectural reference for HitDetectionCommand
---

# HitDetectionCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug
**Type:** Transient Handler

## Definition
```java
// Signature
public class HitDetectionCommand extends CommandBase {
```

## Architecture & Concepts
The **HitDetectionCommand** is a server-side, developer-focused command that provides a runtime toggle for a global debug flag. It embodies the Command Pattern, encapsulating a single, specific action—modifying the visual debug state of the server's entity selection and interaction system.

Architecturally, this class serves as a simple bridge between the high-level Command System and a low-level engine module, **SelectInteraction**. It exposes a mutable, static configuration variable from that module to server administrators via the chat console. Its existence is purely for debugging and development; it has no role in standard gameplay logic.

The command's primary design characteristic is its direct manipulation of a global static field, **SelectInteraction.SHOW_VISUAL_DEBUG**. This is a common, albeit hazardous, pattern for debug flags as it provides immediate, system-wide effect without complex event propagation.

## Lifecycle & Ownership
- **Creation:** An instance of **HitDetectionCommand** is created by the server's central **CommandRegistry** during the server bootstrap phase. The system likely scans the classpath for all subclasses of **CommandBase** and instantiates them for registration.
- **Scope:** The object instance persists for the entire server session, held as a reference within the **CommandRegistry**.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server shuts down and the **CommandRegistry** is cleared.

## Internal State & Concurrency
- **State:** The **HitDetectionCommand** class is entirely **stateless**. It contains no member variables and does not cache any data. Its sole function is to read and modify the state of the external **SelectInteraction** class.

- **Thread Safety:** The class itself is thread-safe due to its stateless nature. However, its primary method, **executeSync**, is designed to be invoked exclusively by the server's main game thread. The name itself is a strong convention indicating it must run synchronously with the game tick to prevent race conditions.

    **WARNING:** The static field it modifies, **SelectInteraction.SHOW_VISUAL_DEBUG**, is a point of contention. Any system reading this flag from an asynchronous thread must implement its own synchronization or risk observing stale data. The engine mitigates this by ensuring both the command's execution and the interaction system's logic run on the same main thread.

## API Surface
The public API is minimal, intended for use only by the command system framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| HitDetectionCommand() | constructor | O(1) | Instantiates the command. For framework use only. |
| executeSync(CommandContext) | void | O(1) | Toggles the global debug flag and notifies the command sender. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically discovered and managed by the server's command processing pipeline. A user or administrator triggers its execution by typing the command into the game console.

The framework performs the following steps:
1. Parses user input (e.g., "/hitdetection").
2. Looks up the corresponding **HitDetectionCommand** instance in the **CommandRegistry**.
3. Invokes the **executeSync** method, passing the current execution context.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new HitDetectionCommand()`. The command system manages the lifecycle. Manually creating an instance will result in an un-registered command that is never executed.
- **Manual Invocation:** Avoid calling `command.executeSync()` directly. Bypassing the central command dispatcher ignores permission checks, argument parsing, and other critical framework features.
- **State Assumption:** Do not write gameplay logic that depends on the value of **SelectInteraction.SHOW_VISUAL_DEBUG**. This is a volatile debug flag that can be changed at any time and may be removed in future versions.

## Data Pipeline
The execution of this command follows a clear, one-way flow from user input to a global state change.

> Flow:
> User Console Input -> Network Packet -> Server Command Parser -> **HitDetectionCommand.executeSync()** -> Global State Change (`SelectInteraction.SHOW_VISUAL_DEBUG`) -> Interaction System Tick (reads new state)

