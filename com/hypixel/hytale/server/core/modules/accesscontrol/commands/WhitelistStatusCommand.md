---
description: Architectural reference for WhitelistStatusCommand
---

# WhitelistStatusCommand

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.commands
**Type:** Transient

## Definition
```java
// Signature
public class WhitelistStatusCommand extends CommandBase {
```

## Architecture & Concepts
The WhitelistStatusCommand is a concrete implementation of the Command Pattern, designed to operate within the server's command processing system. It serves as a terminal endpoint in the command tree, specifically handling the `status` subcommand for the primary `whitelist` command.

Architecturally, this class acts as a simple adapter between the user-facing command system and the server's internal access control logic. Its sole responsibility is to query the current state of the whitelist via the HytaleWhitelistProvider and translate that boolean state into a user-friendly, localized message. It does not contain any business logic, state management, or configuration; it is a stateless handler that delegates all substantive work to the injected provider.

This separation of concerns is critical: the command system handles input parsing and routing, the WhitelistStatusCommand handles the specific `status` request, and the HytaleWhitelistProvider encapsulates the actual state and logic of the whitelist feature.

### Lifecycle & Ownership
- **Creation:** An instance of WhitelistStatusCommand is created during the server's module initialization phase. The parent module responsible for access control (or a dedicated command registration service) instantiates this class, injecting its required HytaleWhitelistProvider dependency via the constructor.
- **Scope:** The object's lifetime is bound to the command's registration. It is created once upon server startup and persists in memory, held by the central command registry, until the server shuts down or the parent module is dynamically unloaded.
- **Destruction:** The instance is eligible for garbage collection when the server's command registry is cleared during the shutdown sequence. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** This class is effectively immutable. Its only internal state is a final reference to the HytaleWhitelistProvider, which is set at construction time. It does not cache data or maintain any mutable fields.

- **Thread Safety:** The class is not inherently thread-safe, nor does it need to be. The `executeSync` method, as dictated by the CommandBase framework, is guaranteed to be invoked on the main server thread. This design choice centralizes state management and prevents race conditions with game world data. Any attempts to invoke its methods from other threads will lead to undefined behavior and likely data corruption.

## API Surface
The public contract is defined by the CommandBase superclass. The primary entry point for the command system is `executeSync`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WhitelistStatusCommand(provider) | constructor | O(1) | Constructs the command. Requires a non-null HytaleWhitelistProvider. |
| executeSync(context) | protected void | O(1) | Executes the command logic. Fetches whitelist status and sends a response to the context's source. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. Instead, it is instantiated and registered with the server's command system during initialization. The correct pattern is to register it as a subcommand of a parent command handler.

```java
// Example from a module's initialization routine
HytaleWhitelistProvider provider = serviceRegistry.get(HytaleWhitelistProvider.class);
CommandManager commandManager = serviceRegistry.get(CommandManager.class);

// Assume a "whitelist" parent command already exists
CommandBase parentWhitelistCommand = commandManager.getCommand("whitelist");
parentWhitelistCommand.registerSubCommand(new WhitelistStatusCommand(provider));
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Never call `executeSync` directly. The command framework is responsible for creating the CommandContext and managing the execution lifecycle. Bypassing the framework will result in a non-functional command.
- **Stateful Implementation:** Do not modify this class to hold mutable state (e.g., caching the last known status). Command objects should remain stateless handlers to ensure predictable behavior across multiple executions.
- **Incorrect Dependency Injection:** Providing a null HytaleWhitelistProvider will result in a NullPointerException at construction. The dependency is non-optional.

## Data Pipeline
The flow of data for a typical `whitelist status` command execution is linear and synchronous, orchestrated by the server's command dispatcher.

> Flow:
> User Input (`/whitelist status`) -> Network Packet -> Command Parser -> **WhitelistStatusCommand**.executeSync -> HytaleWhitelistProvider.isEnabled -> MessageFormat -> CommandContext.sendMessage -> Network Packet -> Client Chat UI

