---
description: Architectural reference for WhitelistDisableCommand
---

# WhitelistDisableCommand

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.commands
**Type:** Transient

## Definition
```java
// Signature
public class WhitelistDisableCommand extends CommandBase {
```

## Architecture & Concepts
The WhitelistDisableCommand is a concrete implementation of the Command Pattern, designed to handle a specific server administration task: disabling the player whitelist. It resides within the server's access control module and serves as the direct link between a user-initiated command string (e.g., `/whitelist off`) and the core service responsible for managing whitelist state.

This class does not maintain any state regarding the whitelist itself. Instead, it acts as a stateless controller that delegates all state modification and queries to an injected **HytaleWhitelistProvider**. This design adheres to the Single Responsibility Principle, cleanly separating the command parsing and execution logic from the underlying business logic of the whitelist system. Its primary role is to validate the current state and trigger the appropriate action on the provider, then report the outcome back to the command issuer.

### Lifecycle & Ownership
- **Creation:** An instance of WhitelistDisableCommand is created during the server's command registration phase. The responsible component, likely a CommandRegistry or a module loader, instantiates this class and injects its required **HytaleWhitelistProvider** dependency.
- **Scope:** The object's lifecycle is tied to the server session. Once registered with the command system, it persists in memory, ready to be executed, until the server shuts down or the command is explicitly unregistered.
- **Destruction:** The instance is eligible for garbage collection when the server shuts down and the central CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its only instance field, whitelistProvider, is a final reference to an external service. All other fields are static, final message templates. It performs no caching and holds no mutable data between executions.
- **Thread Safety:** The core logic is contained within the executeSync method. The method's name explicitly signals that it is designed to be invoked on a synchronized, main server thread. The command system guarantees this execution context. Due to its stateless design, the class is inherently thread-safe, but direct invocation from asynchronous threads is an anti-pattern and would bypass the engine's concurrency model.

## API Surface
The public API is minimal and intended for consumption by the command system, not by general application code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(CommandContext) | void | O(1) | Executes the command logic. Disables the whitelist if it is currently enabled and sends a confirmation message. If already disabled, it sends a notification of the redundant state. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by developers. It is automatically discovered and invoked by the server's command dispatch system in response to player or console input. The system provides the necessary CommandContext for execution.

The *registration* of the command is the primary integration point for a developer extending the system:

```java
// Example: Registering the command within a module's startup routine
HytaleWhitelistProvider provider = serviceRegistry.get(HytaleWhitelistProvider.class);
CommandRegistry commandRegistry = serviceRegistry.get(CommandRegistry.class);

commandRegistry.register(new WhitelistDisableCommand(provider));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation without a Provider:** Never instantiate this class with a null provider, as in `new WhitelistDisableCommand(null)`. This will result in a NullPointerException during execution. The class must be constructed by a system that can supply a valid HytaleWhitelistProvider instance.
- **Direct Invocation:** Do not call the `executeSync` method directly. Bypassing the server's command system will skip critical operations such as permission checks, argument parsing, and thread synchronization, potentially leading to data corruption or an inconsistent server state.

## Data Pipeline
The flow for this command is initiated by an external user action and results in a state change within a core server service.

> Flow:
> User Input (`/whitelist disable`) -> Network Layer -> Command Parser -> Command Dispatcher -> **WhitelistDisableCommand.executeSync()** -> HytaleWhitelistProvider.setEnabled(false) -> CommandContext.sendMessage() -> Network Layer -> User Feedback

