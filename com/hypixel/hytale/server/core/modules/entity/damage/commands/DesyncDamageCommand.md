---
description: Architectural reference for DesyncDamageCommand
---

# DesyncDamageCommand

**Package:** com.hypixel.hytale.server.core.modules.entity.damage.commands
**Type:** Transient Component

## Definition
```java
// Signature
public class DesyncDamageCommand extends CommandBase {
```

## Architecture & Concepts
The DesyncDamageCommand is a server-side administrative command that provides a runtime toggle for a low-level debugging feature within the entity damage system. It serves as a direct control surface, exposing a global static flag to server operators for testing and diagnostic purposes.

Architecturally, this class is an implementation of the Command Pattern. It is discovered and managed by the server's central command system. Its sole responsibility is to receive an execution context and mutate a specific boolean flag, `DamageSystems.FilterUnkillable.CAUSE_DESYNC`. This flag is likely used to intentionally introduce or prevent a state desynchronization between the client and server when an entity that cannot be killed receives damage, allowing developers to test the engine's handling of such edge cases.

This command is not intended for use in production gameplay logic; it is a tool for debugging network and simulation consistency.

### Lifecycle & Ownership
- **Creation:** A single instance of DesyncDamageCommand is created by the server's command registration system during server bootstrap. The system likely scans the classpath for all subclasses of CommandBase and instantiates them.
- **Scope:** The object instance persists for the entire duration of the server session, held as a reference within the central command registry.
- **Destruction:** The instance is eligible for garbage collection upon server shutdown when the command registry is cleared.

## Internal State & Concurrency
- **State:** The DesyncDamageCommand class itself is stateless. It contains no mutable instance fields. However, its primary function is to mutate external, global static state located at `DamageSystems.FilterUnkillable.CAUSE_DESYNC`. This is a significant design choice, indicating its role as a direct manipulator of a global debug flag.

- **Thread Safety:** The `executeSync` method is guaranteed by the command system to be invoked on the main server thread. Therefore, the operations within the method are synchronized with the primary game loop.

    **WARNING:** The static field this command modifies, `CAUSE_DESYNC`, is a global variable. While this command's access is synchronized, any other system accessing this flag from an asynchronous thread would create a race condition. All systems interacting with this flag must do so from the main server thread to ensure visibility and atomicity.

## API Surface
The public contract is defined by its role as a command and is not intended for direct programmatic invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DesyncDamageCommand() | constructor | O(1) | Constructs the command instance and registers its name ("desyncdamage") and description with the base class. |
| executeSync(context) | void | O(1) | Toggles the global `CAUSE_DESYNC` flag and sends a translatable feedback message to the command issuer. |

## Integration Patterns

### Standard Usage
This class is not designed to be used directly in code. It is invoked by the server's command handler when a privileged user types the command into the server console or chat.

**Example (User Input):**
```
/desyncdamage
```
The server command parser identifies "desyncdamage", retrieves the registered DesyncDamageCommand instance, and calls its `executeSync` method, passing in the appropriate CommandContext.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DesyncDamageCommand()`. The command system handles instantiation and registration. Creating an instance manually will result in an object that is not registered and will never be executed.
- **Programmatic Invocation:** Avoid retrieving this command from the registry to call `executeSync` directly. This bypasses the intended user-facing command flow and tightly couples your system to a debug implementation detail.
- **Logic Dependency:** Never write gameplay features that check the state of `DamageSystems.FilterUnkillable.CAUSE_DESYNC`. This is a volatile debug flag that can be changed at any time by a server operator and must not influence production game mechanics.

## Data Pipeline
The data flow for this command is initiated by a user and results in the mutation of a global server state.

> Flow:
> User Input (`/desyncdamage`) -> Network Layer -> Server Command Parser -> **DesyncDamageCommand.executeSync()** -> Mutates `DamageSystems.FilterUnkillable.CAUSE_DESYNC`

A secondary feedback flow is also generated:

> Flow:
> **DesyncDamageCommand.executeSync()** -> CommandContext.sendMessage() -> Server Message System -> Network Layer -> Client Chat UI

