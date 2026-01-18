---
description: Architectural reference for SpawnSuppressionCommand
---

# SpawnSuppressionCommand

**Package:** com.hypixel.hytale.server.spawning.commands
**Type:** Utility

## Definition
```java
// Signature
public class SpawnSuppressionCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The SpawnSuppressionCommand class serves as an administrative entry point for debugging and manipulating the server's spawn suppression system. It is not a persistent service but a command handler that plugs into the server's core command processing system.

Architecturally, this class acts as a high-level bridge between a server administrator and the low-level state managed by the SpawnSuppressionController and the Entity Component System (ECS). It achieves this by registering a collection of sub-commands (add, dump, dumpall) under the primary `suppression` command.

-   The **Add** sub-command is a "write" operation that directly injects a new entity into the world's EntityStore. This new entity is configured with specific components, such as SpawnSuppressionComponent and TransformComponent, which mark it as an active spawn suppressor. Other game systems, specifically the SpawningPlugin, will then detect this entity and incorporate its influence into the mob spawning logic.
-   The **Dump** and **DumpAll** sub-commands are "read" operations used for diagnostics. They query the SpawnSuppressionController, which holds the aggregated and processed state of all spawn suppressors in a world, and format this data into a human-readable report sent to the server logs.

This class follows the Command pattern, encapsulating all information needed to perform an action or trigger an event. It decouples the command invoker (the server's command system) from the object that knows how to perform the action (the World and its EntityStore).

### Lifecycle & Ownership
-   **Creation:** An instance of SpawnSuppressionCommand is created by the SpawningPlugin during the server's bootstrap or plugin loading phase. It is then registered with the central CommandSystem.
-   **Scope:** The object's lifecycle is tied to the SpawningPlugin. It persists for the entire server session.
-   **Destruction:** The command registration is removed and the object is eligible for garbage collection when the SpawningPlugin is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** This class is effectively stateless and immutable after construction. Its primary role is to delegate actions to its inner command classes. All mutable state related to spawn suppression is managed externally by the world's EntityStore and the world-scoped SpawnSuppressionController resource.

-   **Thread Safety:** This class is not thread-safe and is designed to be accessed only from the main server thread. The server's CommandSystem guarantees that all command execution occurs sequentially on this thread, preventing race conditions.

    **Warning:** Calling any `execute` method from an asynchronous task or a different thread will lead to state corruption and server instability. All interactions must be dispatched through the official CommandSystem.

## API Surface
The public contract of this class is not a set of methods but the in-game commands it registers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| suppression add | Command | O(1) | Creates a new spawn suppression entity at the player's current location. Requires a valid SpawnSuppression asset ID. |
| suppression dump | Command | O(N+M) | Dumps the processed spawn suppression data for the current world to the server log. N = suppressors, M = annotated chunks. |
| suppression dumpall | Command | O(W * (N+M)) | Dumps the processed spawn suppression data for all active worlds to the server log. W = number of worlds. |

## Integration Patterns

### Standard Usage
The primary interaction with this class is through in-game commands executed by a server administrator. The following code example illustrates the core logic performed by the `add` sub-command when it is executed.

```java
// This logic is executed inside the 'add' command handler
// It demonstrates how a new suppressor entity is constructed and added to the world

// 1. Resolve arguments from the command context
SpawnSuppression spawnSuppression = this.suppressionArg.get(context);
TransformComponent playerTransform = store.getComponent(ref, TransformComponent.getComponentType());

// 2. Create a new entity holder
Holder<EntityStore> holder = EntityStore.REGISTRY.newHolder();

// 3. Add the necessary components to define a spawn suppressor
holder.addComponent(SpawnSuppressionComponent.getComponentType(), new SpawnSuppressionComponent(spawnSuppression.getId()));
holder.addComponent(TransformComponent.getComponentType(), new TransformComponent(playerTransform.getPosition(), playerTransform.getRotation()));
holder.addComponent(ModelComponent.getComponentType(), new ModelComponent(SpawningPlugin.get().getSpawnMarkerModel()));
holder.addComponent(PersistentModel.getComponentType(), new PersistentModel(SpawningPlugin.get().getSpawnMarkerModel().toReference()));

// 4. Add entity to the world
store.addEntity(holder, AddReason.SPAWN);

// 5. Notify the player
context.sendMessage(Message.translation("server.commands.spawning.suppression.add.added"));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new SpawnSuppressionCommand()` in plugin code. The command collection is managed entirely by the SpawningPlugin's lifecycle. Attempting to register it manually will cause conflicts in the command system.
-   **Direct Execution:** Do not acquire a reference to this object and call the `execute` methods on its inner classes directly. This bypasses critical infrastructure for argument parsing, permission checking, and thread safety provided by the server's CommandSystem.

## Data Pipeline
The data flow differs significantly between the "write" (add) and "read" (dump) operations.

**Add Command Flow:**
> Administrator Input (`/suppression add...`) -> Server Command System -> **SpawnSuppressionCommand.Add** -> EntityStore.addEntity -> World State Change -> SpawnSuppressionController detects and processes the new entity

**Dump Command Flow:**
> Administrator Input (`/suppression dump`) -> Server Command System -> **SpawnSuppressionCommand.Dump** -> SpawnSuppressionController (Read State) -> HytaleLogger -> Server Log File

