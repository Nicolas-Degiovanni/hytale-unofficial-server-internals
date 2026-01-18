---
description: Architectural reference for WhitelistClearCommand
---

# WhitelistClearCommand

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.commands
**Type:** Command Object

## Definition
```java
// Signature
public class WhitelistClearCommand extends CommandBase {
```

## Architecture & Concepts
The WhitelistClearCommand is a concrete implementation within the server's Command System. It encapsulates the logic for a single, specific administrative action: clearing all entries from the server's whitelist.

Architecturally, this class serves as a translation layer between a user-initiated command string and a direct, state-mutating operation on the access control module. It decouples the command parsing and routing infrastructure, handled by the parent CommandBase and the wider system, from the business logic of whitelist management. Its sole responsibility is to interact with the HytaleWhitelistProvider to perform the clear operation and report the result back to the command issuer.

This pattern ensures that high-level server administration tasks are exposed through a consistent and secure command interface, rather than requiring direct API calls into sensitive modules.

## Lifecycle & Ownership
- **Creation:** An instance of WhitelistClearCommand is created during the server's bootstrap or module initialization phase. The required HytaleWhitelistProvider dependency is injected by a dependency injection container or a factory responsible for constructing and registering all server commands.

- **Scope:** The object's lifetime is tied to the server session. It is instantiated once upon startup and remains registered in the central command dispatcher until the server shuts down.

- **Destruction:** The instance is eligible for garbage collection when the server shuts down and the command registry is cleared. It holds no persistent resources that require manual cleanup.

## Internal State & Concurrency
- **State:** This class is effectively stateless. It holds a single, final reference to the HytaleWhitelistProvider. It does not cache data or contain any mutable fields. All stateful operations are delegated to the injected provider.

- **Thread Safety:** This class is **not** thread-safe for arbitrary invocation. The design contract, indicated by the `executeSync` method name, mandates that it be executed exclusively on the main server thread. This convention prevents race conditions when modifying the underlying whitelist collection. The atomicity of the modification is guaranteed by the `HytaleWhitelistProvider.modify` method, which is expected to handle its own internal locking or thread-safe queuing.

## API Surface
The primary interaction point is the overridden `executeSync` method, which is part of the CommandBase contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(N) | Atomically clears all entries from the whitelist, where N is the number of whitelisted users. Sends a confirmation message to the issuer via the provided CommandContext. This is a blocking operation. |

## Integration Patterns

### Standard Usage
Developers do not typically invoke this class directly. It is registered with the server's command system, which handles its execution in response to user input. The registration process is the standard integration point.

```java
// Example: Registering the command during module initialization
HytaleWhitelistProvider provider = context.getService(HytaleWhitelistProvider.class);
CommandRegistry registry = context.getService(CommandRegistry.class);

// The "whitelist" parent command would then register this as a subcommand
parentWhitelistCommand.addSubCommand(new WhitelistClearCommand(provider));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WhitelistClearCommand()` in general application logic. Command objects are part of the server's core infrastructure and should only be instantiated during the initial setup phase.

- **Manual Execution:** Avoid obtaining a reference to this command and calling `executeSync` directly. This bypasses the command system's permission checks and context setup, and may violate threading guarantees.

- **Asynchronous Invocation:** Calling `executeSync` from a separate worker thread is a severe violation of the class's concurrency contract. All command execution must be dispatched through the server's main tick loop to ensure data integrity.

## Data Pipeline
The flow of data for this command begins with user input and ends with a state change and a confirmation message.

> Flow:
> User Input (`/whitelist clear`) -> Command Parser -> Command Dispatcher -> **WhitelistClearCommand.executeSync()** -> HytaleWhitelistProvider.modify() -> Whitelist Cleared -> CommandContext.sendMessage() -> Confirmation Message to User

