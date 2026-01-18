---
description: Architectural reference for PrefabEditExitCommand
---

# PrefabEditExitCommand

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabEditExitCommand extends AbstractAsyncPlayerCommand {
```

## Architecture & Concepts
The PrefabEditExitCommand class is an implementation of the Command Pattern, designed to integrate with the Hytale server's core command processing system. It serves as the player-facing entry point for terminating a prefab editing session.

This command's primary architectural role is that of an **Orchestrator**. It does not contain the logic for session management itself. Instead, it validates the player's state and delegates the complex teardown process to the authoritative PrefabEditSessionManager.

A key feature of its design is the handling of unsaved work. If the session is "dirty", the command does not force an exit. It transitions the player's state by opening a confirmation UI, effectively handing off control to the UI system. This decouples the initial user intent (exiting) from the subsequent, state-dependent actions (saving, discarding, or canceling), creating a more robust and user-friendly workflow.

## Lifecycle & Ownership
- **Creation:** An instance of PrefabEditExitCommand is created by the server's command registry during the dispatch process for a single player command. It is not a long-lived object.

- **Scope:** The object's lifetime is exceptionally short, scoped exclusively to the execution of one command. Once the `executeAsync` method returns its CompletableFuture, the instance has served its purpose and is eligible for garbage collection.

- **Destruction:** The Java Garbage Collector is responsible for destruction. There is no manual cleanup or `close` method, as the object is stateless.

## Internal State & Concurrency
- **State:** PrefabEditExitCommand is fundamentally stateless. It maintains no instance-level fields that track session data. All required context, such as the player, world, and entity stores, is provided as method parameters by the command system during invocation.

- **Thread Safety:** The class itself is inherently thread-safe due to its stateless design. However, it operates within a concurrent environment. The `executeAsync` method is designed to be called by the server's asynchronous task scheduler. All interactions with shared systems, particularly the PrefabEditSessionManager, must be handled in a thread-safe manner by those systems. This command assumes that the session manager correctly synchronizes its internal state.

## API Surface
The public API is minimal, consisting of the constructor used by the command system and the overridden execution method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(...) | CompletableFuture<Void> | O(N) | Orchestrates the session exit. Complexity is O(N) where N is the number of prefabs in the session, due to the linear scan for dirty metadata. |

## Integration Patterns

### Standard Usage
This class is not designed for direct programmatic invocation. It is automatically discovered and registered by the server's command system. A developer's interaction is limited to ensuring the BuilderToolsPlugin is enabled. The system is triggered by a player executing the command in-game.

```java
// This class is executed by the server's command system.
// A player triggers it by typing the associated command in chat.
// Example: /prefab edit exit

// The system handles the instantiation and invocation internally.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance via `new PrefabEditExitCommand()`. The object is useless without the context (PlayerRef, World, Store) provided by the command system's `executeAsync` call.

- **Programmatic Invocation:** Do not attempt to call the `executeAsync` method directly. This bypasses the server's permission checks, context injection, and thread management, which can lead to instability and race conditions.

- **Adding State:** Do not modify this class to include instance fields for storing data. The command system provides no guarantee that the same instance will be used for multiple commands, even from the same player.

## Data Pipeline
This component acts as a control-flow trigger rather than a data processor. Its "pipeline" represents a sequence of operations and state transitions across multiple systems.

> Flow:
> Player Command Input -> Server Command System -> **PrefabEditExitCommand.executeAsync** -> PrefabEditSessionManager.isEditingAPrefab -> PrefabEditSession.getLoadedPrefabMetadata -> (Conditional) Player.getPageManager.openCustomPage -> (Default) PrefabEditSessionManager.exitEditSession

