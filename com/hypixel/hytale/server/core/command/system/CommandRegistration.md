---
description: Architectural reference for CommandRegistration
---

# CommandRegistration

**Package:** com.hypixel.hytale.server.core.command.system
**Type:** Transient

## Definition
```java
// Signature
public class CommandRegistration extends Registration {
```

## Architecture & Concepts
The CommandRegistration class is a foundational component of the server's command system. It is not a service or manager, but rather a **handle** or **record** that represents a single, registered command. Its primary architectural purpose is to decouple a command's implementation (the AbstractCommand) from its lifecycle management.

This class encapsulates three key pieces of information:
1.  The command logic itself (an instance of AbstractCommand).
2.  A condition for its availability (a BooleanSupplier).
3.  A procedure for its removal (a Runnable).

By bundling these together, the central CommandSystem can manage a heterogeneous collection of commands without needing to know the specific enabling conditions or unregistration logic for each one. This is a critical pattern for modularity, allowing plugins or other game systems to register commands and retain full control over their lifetime. The CommandRegistration object acts as the contract between the command provider and the command system.

It inherits from the base Registration class, indicating a standardized pattern for managing dynamically registered entities throughout the engine.

## Lifecycle & Ownership
-   **Creation:** A CommandRegistration instance is created exclusively by the core command management service (e.g., CommandSystem) when external code calls a method like `registerCommand`. The system takes the raw command and its lifecycle delegates and wraps them in this object.
-   **Scope:** The object's lifetime is precisely tied to the duration of the command's registration. It is held within a private collection inside the CommandSystem and persists as long as the command is active and available to be executed.
-   **Destruction:** The object is marked for garbage collection when the command is unregistered. The `unregister` Runnable, provided at creation, is invoked, which typically removes the CommandRegistration instance from the CommandSystem's internal registry. Once all external references to this handle are released, it is fully destroyed.

## Internal State & Concurrency
-   **State:** The internal state of a CommandRegistration object is **immutable**. The reference to the AbstractCommand and the lifecycle delegates (BooleanSupplier, Runnable) are marked as final and are set only once during construction. This design guarantees that a registration handle cannot be modified after it has been created.

-   **Thread Safety:** The CommandRegistration object itself is inherently thread-safe due to its immutability. However, this guarantee does not extend to the delegates it holds. The provided BooleanSupplier and Runnable are external code. It is the responsibility of the *provider* of these delegates to ensure they are thread-safe if the command system operates in a multi-threaded context.

    **Warning:** The CommandSystem must synchronize access to the `isEnabled` and `unregister` delegates if they can be invoked from multiple threads. Race conditions within the delegate logic can lead to system instability.

## API Surface
The public contract is primarily defined by the parent Registration class and an assumed accessor for the command.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCommand() | AbstractCommand | O(1) | (Assumed) Returns the underlying command instance. |
| isEnabled() | boolean | O(N) | (Inherited) Invokes the BooleanSupplier. Complexity depends on the delegate's implementation. |
| unregister() | void | O(N) | (Inherited) Invokes the Runnable to tear down the registration. Complexity depends on the delegate's implementation. |

## Integration Patterns

### Standard Usage
A developer typically does not interact with this class directly. Instead, they receive it as a return value from a registration call, which can be used later to manually unregister the command.

```java
// A system receives a handle after registering a command.
// This handle can be stored if manual unregistration is required.

CommandSystem commandSystem = context.getService(CommandSystem.class);
MyCommand myCommand = new MyCommand();

CommandRegistration registrationHandle = commandSystem.register(
    myCommand,
    () -> myPlugin.isEnabled(), // The command is only enabled if its plugin is.
    () -> log.info("MyCommand has been unregistered.")
);

// Later, to clean up...
registrationHandle.unregister();
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new CommandRegistration()`. This object is meaningless outside of the CommandSystem that manages it. Doing so will result in a command that is not registered and cannot be executed.
-   **Caching Enabled State:** Do not call `isEnabled()` once and cache the result. The BooleanSupplier is designed to be evaluated dynamically each time the command's availability is checked. Caching its state can lead to commands being available when they should be disabled (e.g., after a plugin is disabled).
-   **Retaining Handles Post-Unregister:** Do not hold a reference to a CommandRegistration object after `unregister()` has been called. The handle is considered stale and any further interaction with it is undefined behavior.

## Data Pipeline
CommandRegistration is a structural component, not a data processing one. It functions as a container within the command registration and unregistration flow.

> **Registration Flow:**
> Plugin/Module Code -> `CommandSystem.register(command, delegates)` -> **new CommandRegistration(command, delegates)** -> Stored in CommandSystem Registry

> **Execution Flow:**
> Player Input -> Command Parser -> `CommandSystem.findCommand(name)` -> Accesses **CommandRegistration** -> `isEnabled()` check -> `getCommand().execute()`

