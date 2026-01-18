---
description: Architectural reference for AuthPersistenceCommand
---

# AuthPersistenceCommand

**Package:** com.hypixel.hytale.server.core.command.commands.server.auth
**Type:** Transient

## Definition
```java
// Signature
public class AuthPersistenceCommand extends CommandBase {
```

## Architecture & Concepts
The AuthPersistenceCommand serves as a high-level administrative interface for managing the server's authentication credential storage strategy. It is a concrete implementation of the Command Pattern, designed to be discovered and executed by the server's central command processing system.

This command does not implement any authentication logic itself. Instead, it acts as a controller, dispatching instructions to core, stateful singletons. Its primary responsibility is to query and modify the active AuthCredentialStoreProvider, which dictates how server authentication tokens are persisted (e.g., in-memory, to a file, or not at all).

Interaction with different storage providers is decoupled via the AuthCredentialStoreProvider.CODEC, a BuilderCodec instance. This allows the system to be extended with new persistence mechanisms without modifying this command. The command dynamically queries the codec for available provider types and uses it to instantiate a new provider when requested by an administrator.

## Lifecycle & Ownership
-   **Creation:** A single instance of AuthPersistenceCommand is created by the command registration system during server bootstrap. It is then registered with the central command handler under the name *persistence* within the *auth* command group.
-   **Scope:** The object instance persists for the entire lifetime of the server process. However, its execution is ephemeral; the `executeSync` method is invoked for a single command execution and does not retain state afterward.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection during the server shutdown sequence when the command registry is cleared.

## Internal State & Concurrency
-   **State:** AuthPersistenceCommand is fundamentally stateless. It holds no mutable instance fields. All information it presents or modifies is retrieved from and written to external, authoritative sources, namely the HytaleServer configuration and the ServerAuthManager. The static Message fields are immutable, pre-configured constants for user feedback.
-   **Thread Safety:** This class is not thread-safe and must not be treated as such. The `executeSync` method is designed to be invoked exclusively by the server's main thread as part of the primary game loop tick. Any attempt to invoke it from an asynchronous context will bypass server-wide state management locks and lead to severe data corruption and race conditions within the ServerAuthManager and server configuration.

## API Surface
The primary contract is not a traditional method API but its registration with the command system. The `executeSync` method is the entry point for the command handler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(N) | Executes the command logic. If no arguments are provided, it reports the current and available persistence providers (Complexity O(N) where N is the number of registered providers). If a provider type is given via the SetPersistenceVariant, it changes the server's active provider (Complexity O(1)). |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically executed by the server's command handler in response to administrator input from the console. The example below illustrates how the command system would interact with the class.

```java
// This logic resides within the server's command handler.
// A developer would not write this code.

// 1. The handler parses console input "auth persistence file"
CommandContext context = buildContextFromInput("auth persistence file");

// 2. The handler retrieves the registered command instance
AuthPersistenceCommand command = commandRegistry.get("auth persistence");

// 3. The handler invokes the command's execution logic
command.executeSync(context);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new AuthPersistenceCommand()`. An un-registered command instance is inert and will never be executed by the server.
-   **Asynchronous Execution:** Never call `executeSync` from a separate thread. Modifying the `AuthCredentialStoreProvider` is a critical operation that must be synchronized with the main server tick to prevent race conditions in the ServerAuthManager.
-   **State Assumption:** Do not attempt to store state on an instance of this class. The command system guarantees a single instance, but its use as a state container is an abuse of the design and will lead to unpredictable behavior.

## Data Pipeline
AuthPersistenceCommand acts as a control point, not a data processing pipeline. The flow represents a user-initiated command-and-control sequence.

> Flow:
> Administrator Console Input -> Command Parser -> **AuthPersistenceCommand.executeSync** -> HytaleServer.Config (Update) -> ServerAuthManager.swapCredentialStoreProvider -> New AuthCredentialStoreProvider (Instantiation)

