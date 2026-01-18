---
description: Architectural reference for TimerFragmentCommand
---

# TimerFragmentCommand

**Package:** com.hypixel.hytale.builtin.portals.commands
**Type:** Transient

## Definition
```java
// Signature
public class TimerFragmentCommand extends PortalWorldCommandBase {
```

## Architecture & Concepts
The TimerFragmentCommand is a concrete implementation of the Command Pattern, designed to integrate seamlessly into the server's command processing system. Its sole responsibility is to expose a specific administrative action—modifying the remaining time of a Portal World—to an authorized user or system process via a text-based command interface.

Architecturally, this class functions as a thin adapter layer. It translates a structured command invocation, complete with parsed arguments, into a direct state change on a game world entity, specifically the PortalWorld component. It inherits from PortalWorldCommandBase, a framework class that abstracts away the boilerplate logic of validating the command's execution context. This base class ensures that the command is only run within a valid PortalWorld and provides the `execute` method with pre-validated, non-null world objects, enforcing a clear separation of concerns.

This command is not a long-lived service or manager; it is a stateless, single-purpose handler whose existence is primarily for registration with the central Command System.

### Lifecycle & Ownership
- **Creation:** A single instance of TimerFragmentCommand is instantiated by the server's command registration system during the server bootstrap or module loading phase. It is discovered and registered to handle the "timer" command string.
- **Scope:** The singleton instance persists for the entire server session. It remains in memory, ready to be invoked by the command dispatcher whenever a matching command is entered.
- **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down or the associated game module is unloaded, at which point the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. The `remainingSecondsArg` field is a final, immutable definition of a command argument, not a container for runtime data. All state modifications are performed on external objects (`PortalWorld`, `World`) that are passed into the `execute` method.
- **Thread Safety:** This class is not thread-safe and is not designed to be. The server's command execution framework is responsible for ensuring that command logic is executed on the appropriate world's main thread. Invoking the `execute` method from an external, unmanaged thread will lead to race conditions and severe world state corruption. All interactions with the `World` and `PortalWorld` objects must be synchronized with the main game loop.

## API Surface
The public contract is minimal, intended for use only by the command system framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, portalWorld, store) | void | O(1) | Executes the core command logic. Modifies the PortalWorld's remaining time and sends a confirmation message to the command source. |

## Integration Patterns

### Standard Usage
A developer or administrator does not interact with this class directly in Java code. The standard usage is to invoke the command through a server-supported text interface, such as the server console or in-game chat.

```sh
# Sets the remaining time for the current portal world to 300 seconds.
/timer 300
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class using `new TimerFragmentCommand()`. It is useless outside the context of the command registration system and provides no functional value.
- **Manual Execution:** Do not call the `execute` method directly. This bypasses the entire command processing framework, including critical steps like permission checks, context validation, and argument parsing. Manual invocation requires constructing complex state objects (`CommandContext`, `World`) that are managed exclusively by the server engine.

## Data Pipeline
The flow of data for this command is unidirectional, originating from user input and resulting in a world state change and a feedback message.

> Flow:
> User Input (`/timer 300`) -> Server Command Parser -> Command Dispatcher -> **TimerFragmentCommand.execute()** -> PortalWorld.setRemainingSeconds() -> CommandContext.sendMessage() -> User Feedback Message

