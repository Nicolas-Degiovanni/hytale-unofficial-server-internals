---
description: Architectural reference for PacksListCommand
---

# PacksListCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.packs
**Type:** Transient

## Definition
```java
// Signature
public class PacksListCommand extends CommandBase {
```

## Architecture & Concepts
PacksListCommand is a concrete implementation of the Command Pattern, designed for server administration and debugging. It functions as a read-only bridge between the server's command-line interface and the core asset management system.

Its primary architectural role is to decouple the command issuer (e.g., a server administrator) from the internal state of the AssetModule. By encapsulating the logic for querying and formatting asset pack information, it provides a stable, user-facing endpoint that is resilient to changes in the underlying asset system. This command is registered under the parent command *packs* with the name *list* and alias *ls*.

This class is considered a leaf node in the server's command hierarchy and holds no business logic beyond fetching, sorting, and presenting data.

## Lifecycle & Ownership
- **Creation:** A single instance of PacksListCommand is created by the CommandSystem during server bootstrap. The system scans for command classes and instantiates them for registration in a central command registry.
- **Scope:** The object's lifecycle is tied to the server session. It is instantiated once and persists until the server shuts down. It is effectively a singleton managed by the command framework.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the CommandSystem is dismantled during the server shutdown sequence.

## Internal State & Concurrency
- **State:** PacksListCommand is **stateless and immutable**. It contains only static final fields for predefined message templates. It does not cache data or maintain any state between executions. All necessary information is fetched from the AssetModule on each invocation.

- **Thread Safety:** The class is inherently thread-safe due to its stateless design. The primary execution method, executeSync, signals a strict contract: the calling CommandSystem is responsible for invoking it from a synchronized context, typically the main server thread. Direct invocation from asynchronous threads would violate this contract and could lead to race conditions when accessing the AssetModule.

## API Surface
The public contract is minimal and dictated by the CommandBase superclass. The intended interaction is exclusively through the command execution framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(CommandContext) | void | O(N log N) | Fetches, sorts, and displays all registered AssetPacks. The complexity is dominated by the case-insensitive sort of pack names. Throws NullPointerException if context is null. |

## Integration Patterns

### Standard Usage
This class is not designed to be used directly in code. It is invoked by the server's command dispatcher when a privileged user executes the corresponding command in the server console.

The following example illustrates the conceptual flow of how the CommandSystem would delegate a user's input to this command.

```java
// Conceptual: How the CommandSystem dispatches to this command.
// A developer does not write this code.
CommandSystem commandSystem = server.getCommandSystem();
CommandContext context = create_context_for_user("admin");

// User types "/packs list" into the console.
commandSystem.dispatch("packs list", context);

// Internally, the system finds the registered PacksListCommand instance
// and invokes its executeSync method with the prepared context.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new PacksListCommand()`. The object is useless unless registered with the server's CommandSystem, which handles its lifecycle.
- **Manual Invocation:** Do not call the `executeSync` method directly. Bypassing the CommandSystem circumvents critical infrastructure, including permission checks, context population, and thread safety guarantees. This can lead to inconsistent server state or concurrency violations.
- **Stateful Extension:** Do not extend this class to add state. The command framework relies on commands being stateless and reusable.

## Data Pipeline
The data flow for this command is a simple request-response sequence initiated by a user.

> Flow:
> Server Console Input -> Command System Parser -> **PacksListCommand** -> AssetModule Query -> Message Formatter -> Server Console Output

