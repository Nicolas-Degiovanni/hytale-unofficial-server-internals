---
description: Architectural reference for WarpGoCommand
---

# WarpGoCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.warp
**Type:** Command Handler

## Definition
```java
// Signature
public class WarpGoCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The WarpGoCommand class is a concrete implementation of the Command Pattern, designed to integrate specifically with the server's command processing system. It functions as a leaf node in the command hierarchy, responsible for handling the `go` subcommand, typically invoked by players as `/warp go <warpName>`.

Its primary architectural role is to act as a lightweight adapter. It translates a raw player command into a structured, validated call to the core warp system. The class is responsible for three distinct tasks:
1.  **Argument Parsing:** It defines and extracts the required `warpName` string argument from the command context.
2.  **Permission Enforcement:** It ensures the executing player possesses the `hytale.command.warp.go` permission before proceeding.
3.  **Logic Delegation:** It does not contain any teleportation logic itself. Instead, it delegates the entire operation to the static `WarpCommand.tryGo` method, passing along the necessary player and world context. This separation of concerns keeps the command definition clean and centralizes the core warp functionality.

## Lifecycle & Ownership
-   **Creation:** Instantiated once during server bootstrap by the command registration system. It is typically registered as a subcommand to a parent `WarpCommand`.
-   **Scope:** Singleton-like for the duration of the server's runtime. A single instance handles all `/warp go` command invocations.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless and immutable after construction. Its only field, `warpNameArg`, is a final definition of the command's argument structure and does not change during the object's lifetime. The `execute` method operates exclusively on parameters passed into it, not on internal instance state.
-   **Thread Safety:** The class is inherently thread-safe due to its immutable nature. Command execution is managed by the server's main command processing thread, which serializes invocations of the `execute` method. Concurrency concerns, if any, exist within the downstream systems it calls (e.g., `EntityStore`), not within WarpGoCommand itself.

## API Surface
The public contract is fulfilled by overriding the `execute` method from its parent, `AbstractPlayerCommand`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Parses the `warpName` argument and delegates the teleportation attempt to `WarpCommand.tryGo`. Throws command-specific exceptions if argument parsing fails. The complexity of the downstream call is not reflected here. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is designed to be discovered and registered by the server's command system. The conceptual usage pattern involves adding it as a subcommand to a parent command during server initialization.

```java
// Conceptual example of command registration
// This code would exist within the server's bootstrap sequence.

WarpCommand parentWarpCommand = new WarpCommand();
parentWarpCommand.addSubCommand(new WarpGoCommand());

// The command system would then register the parent command.
server.getCommandRegistry().register(parentWarpCommand);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Invocation:** Never instantiate and call the `execute` method manually. Doing so bypasses the command system's context creation, argument parsing, and permission-checking pipeline, leading to unpredictable behavior and NullPointerExceptions.
-   **Stateful Implementation:** Do not add mutable instance fields to this class. Command handlers are expected to be stateless to ensure thread safety and predictable execution across multiple players.

## Data Pipeline
WarpGoCommand acts as a specific processing node in the server's command handling pipeline. It receives a structured context and dispatches a call to a more generalized system.

> Flow:
> Player Chat Input (`/warp go my_base`) → Server Network Layer → Command Parser → **WarpGoCommand.execute()** → Argument Extraction (`my_base`) → `WarpCommand.tryGo()` → EntityStore Lookup & Player State Mutation → Network Position Update Packet → Client Render Update

