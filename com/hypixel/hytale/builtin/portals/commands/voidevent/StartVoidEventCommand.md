---
description: Architectural reference for StartVoidEventCommand
---

# StartVoidEventCommand

**Package:** com.hypixel.hytale.builtin.portals.commands.voidevent
**Type:** Transient

## Definition
```java
// Signature
public class StartVoidEventCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The StartVoidEventCommand is a server-side administrative command responsible for initiating a "Void Event" within a game world. As a subclass of AbstractWorldCommand, it is designed to operate within the context of a specific World instance, ensuring that its actions are localized and world-aware.

Its primary function is to interact with the **PortalWorld** resource, a specialized data component attached to a World's EntityStore. This command acts as a high-level control mechanism, checking the current state of the PortalWorld and, if no event is active, triggering a new one by manipulating its internal timer.

A key architectural feature is its "override" capability, controlled by a command flag. If the target world is not already configured as a PortalWorld, this command can forcibly reconfigure it. This process involves loading hardcoded asset configurations (specifically the *Portal* GameplayConfig and the *Hederas_Lair* PortalType) and initializing a new PortalWorld resource from scratch. This makes the command a powerful tool for both managing existing portal events and for dynamically setting up new ones for testing or administrative purposes.

## Lifecycle & Ownership
-   **Creation:** An instance of StartVoidEventCommand is created and registered by the server's central CommandSystem during the server bootstrap or plugin loading phase.
-   **Scope:** The object instance persists for the entire server session, held within the command registry. However, the context provided to its *execute* method (such as the CommandContext and World) is transient and only valid for the duration of a single command invocation.
-   **Destruction:** The instance is dereferenced and garbage collected when the server shuts down and the CommandSystem is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. Its fields, such as `overrideWorld`, define the command's argument structure and are immutable after construction. All stateful operations performed by the `execute` method target external objects, primarily the `World` and its associated `PortalWorld` resource.
-   **Thread Safety:** This command is not thread-safe and is not designed for concurrent execution. Server commands are executed serially on the main server thread (the world tick thread). Any attempt to invoke the `execute` method from an asynchronous task or a different thread will lead to severe concurrency violations, including race conditions and data corruption within the world state.

## API Surface
The public API is defined by its role as a command and is not intended for direct invocation outside the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| StartVoidEventCommand() | constructor | O(1) | Constructs the command, registering its name ("start"), description key, and the "override" flag argument. |
| execute(context, world, store) | void | Variable | The command's entry point. Modifies the target World and its PortalWorld resource to initiate a Void Event. Complexity depends on asset lookups and resource initialization. |

## Integration Patterns

### Standard Usage
This class is designed to be invoked via the server console or by an in-game entity with sufficient permissions. It is not intended to be used directly in code.

**Console Invocation:**
```sh
# Starts a void event in the current world, if it's a portal world
/voidevent start

# Forcibly configures the current world as a portal world and starts the event
/voidevent start --override
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new StartVoidEventCommand()` in your own code. The command system manages its lifecycle. Creating an instance has no effect as it will not be registered for execution.
-   **Manual Execution:** Never call the `execute` method directly. Doing so bypasses the entire command processing pipeline, including argument parsing, permission checks, and context provision. This can lead to NullPointerExceptions and an unstable server state.
-   **Asset Dependency Assumption:** Do not assume the hardcoded assets "Portal" and "Hederas_Lair" will always exist. While they are part of the builtin module, robust systems should handle potential asset loading failures. This command contains a basic null check but does not implement more advanced fallback logic.

## Data Pipeline
StartVoidEventCommand acts as a control-flow initiator rather than a data processor. Its execution triggers a state change in the world.

> Flow:
> Player/Console Input -> Command Parser -> **StartVoidEventCommand.execute()** -> World State & PortalWorld Resource Mutation -> Event Begins in World

