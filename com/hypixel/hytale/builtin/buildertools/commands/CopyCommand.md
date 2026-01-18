---
description: Architectural reference for CopyCommand
---

# CopyCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Command Handler

## Definition
```java
// Signature
public class CopyCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The **CopyCommand** class serves as the server-side, user-facing entry point for the Builder Tools clipboard functionality. It is a concrete implementation of the command pattern, inheriting from **AbstractPlayerCommand**, which integrates it directly into the server's command processing system.

Its primary architectural role is to act as a translator between player input (the `/copy` command and its arguments) and the core logic encapsulated within the **BuilderToolsPlugin**. It does not perform the copy operation itself. Instead, it parses arguments, validates player state and permissions, and then defers the actual world modification task to a managed execution queue via **BuilderToolsPlugin.addToQueue**.

This delegation is a critical design choice. It decouples the command parsing logic from the potentially long-running and thread-sensitive task of reading a world region into memory. By placing the operation onto a queue, the system ensures that all builder tool operations are executed serially and on the correct server thread, preventing race conditions and world corruption.

The class defines two operational variants:
1.  A parameter-less version that copies the player's current selection region.
2.  A variant implemented by the inner class **CopyRegionCommand** that copies an explicitly defined bounding box from coordinates provided as arguments.

## Lifecycle & Ownership
-   **Creation:** A single instance of **CopyCommand** is instantiated by the server's command registration framework when the **BuilderToolsPlugin** is loaded. The constructor call `super("copy", ...)` registers the command name and its description with the system.
-   **Scope:** The object is a long-lived service. It persists for the entire lifecycle of the server or until the parent plugin is disabled. It is not created on a per-command basis.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down or the command registry is cleared during a plugin unload.

## Internal State & Concurrency
-   **State:** The **CopyCommand** instance itself is effectively stateless with respect to individual command executions. Its member fields, such as **noEntitiesFlag**, are immutable definitions for command arguments, configured once during construction. All state required for an operation (the player, the world, the selection) is passed into the `execute` method as parameters.
-   **Thread Safety:** This class is thread-safe. The `execute` method is invoked by the server's command processing thread pool. Crucially, it avoids direct world interaction. All world-mutating logic is encapsulated within a lambda and passed to **BuilderToolsPlugin.addToQueue**. This pattern ensures that the complex and stateful part of the operation is executed on a dedicated, synchronized thread, making the command handler itself safe for concurrent invocation.

## API Surface
The public API is designed for two distinct callers: the command system and internal server code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(1) | Framework entry point. Invoked by the command system when a player runs `/copy`. Validates and enqueues the copy operation. Complexity is constant as it only submits a task. |
| copySelection(...) | public static void | O(1) | Static utility for programmatic invocation. Allows other systems to trigger a copy of a player's selection without command input. Throws an assertion error if the player entity is missing required components. |

## Integration Patterns

### Standard Usage
The primary integration is through player-initiated commands. The server framework handles parsing and routing to the correct instance.

```java
// This code is never written by a developer.
// It represents a player typing a command in the game client.
// > /copy --noEntities

// The server's command system invokes the registered handler:
copyCommandInstance.execute(context, store, ref, playerRef, world);
```

### Programmatic Usage
The static **copySelection** method provides a decoupled way for other game systems or plugins to trigger the copy functionality, for example, from a custom UI button.

```java
// How a developer should programmatically trigger a copy
// for a given player entity.
Ref<EntityStore> playerEntityRef = ...;
ComponentAccessor<EntityStore> accessor = ...;

CopyCommand.copySelection(playerEntityRef, accessor);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new CopyCommand()`. This creates an orphan object that is not registered with the command system and will never be executed.
-   **Bypassing the Queue:** Do not replicate the logic from the `execute` method's lambda to interact with **BuilderState** directly. All world modifications must go through the **BuilderToolsPlugin.addToQueue** method to ensure thread safety and proper sequencing of operations.
-   **Directly Invoking Execute:** The `execute` method should only be called by the server's command processing framework. Calling it manually may bypass permission checks and context setup.

## Data Pipeline
The flow of data for a standard command execution is unidirectional, moving from player input to the core builder system.

> Flow:
> Player Command (`/copy`) -> Server Command Parser -> **CopyCommand.execute** -> Argument & Flag Parsing -> **BuilderToolsPlugin.addToQueue** -> (On Main Thread) -> **BuilderState.copyOrCut** -> World Region Data Read -> Player Clipboard State Update

