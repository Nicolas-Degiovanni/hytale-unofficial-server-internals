---
description: Architectural reference for ExitInstance
---

# ExitInstance

**Package:** com.hypixel.hytale.builtin.instances.config
**Type:** Configuration-Driven Strategy

## Definition
```java
// Signature
public class ExitInstance implements RespawnController {
```

## Architecture & Concepts
The ExitInstance class is a concrete implementation of the RespawnController strategy interface. Its role is to define a specific behavior for player respawns that occur within a game *instance*.

Unlike traditional respawn handlers that reposition a player within the current world, the primary function of ExitInstance is to remove the player from the instance entirely. It acts as a gateway, triggered by a player's death, to transition them back to their previous location or a designated world.

This component is a critical piece of the server's instancing framework. It decouples the core respawn logic from the specialized rules of temporary game worlds. The class achieves this by delegating the complex task of exiting an instance to the central InstancesPlugin.

A key architectural feature is its built-in fault tolerance. It contains a fallback RespawnController, which is invoked if the primary instance-exit logic fails for any reason. This ensures system stability and prevents players from becoming trapped in a broken state. By default, this fallback attempts to respawn the player at their home or a world spawn point.

## Lifecycle & Ownership
- **Creation:** ExitInstance is not instantiated directly via its constructor in code. It is created by the Hytale codec system during server startup or world loading when parsing game configuration files. The static CODEC field defines the deserialization and construction logic.
- **Scope:** The object's lifetime is bound to the configuration in which it is defined. It persists as long as the server has that specific instance or world configuration loaded in memory.
- **Destruction:** The object is marked for garbage collection when its parent configuration is unloaded. It does not manage any native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The class holds a single stateful field, *fallback*, which is a reference to another RespawnController. This field is set once during deserialization and is effectively immutable thereafter. The class does not cache any runtime data or maintain session-specific state.
- **Thread Safety:** This class is thread-safe. The respawnPlayer method is designed to be called from the main server thread for a given world. It operates exclusively on the arguments passed to it and does not modify its own internal state. The use of a CommandBuffer for all world modifications ensures that entity state changes are queued and executed at a safe point in the server tick, preventing race conditions.

## API Surface
The public API is defined by the RespawnController interface and consists of a single method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| respawnPlayer(world, playerReference, commandBuffer) | void | O(N) | Attempts to exit the player from the current instance. If an exception occurs, it invokes the configured fallback respawn controller. The complexity is dependent on the underlying instance exit logic. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly from procedural code. It is designed to be declared within a data-driven configuration file, such as a world or instance definition. The server's respawn system then automatically selects and executes this controller based on the player's context.

A conceptual configuration might look like this:

```json
// In a hypothetical instance_template.json
"respawnController": {
  "type": "ExitInstance",
  "Fallback": {
    "type": "HomeOrSpawnPoint"
  }
}
```

The game engine would then use the registered CODEC to deserialize this block into an ExitInstance object.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ExitInstance()`. Doing so bypasses the codec system, which is responsible for correctly configuring the essential *fallback* field. An unconfigured instance will fail to handle errors gracefully.
- **Recursive Fallback:** Configuring the *fallback* controller to be another ExitInstance or a controller that could lead back to the same instance can create an infinite loop on repeated failures, potentially crashing the server thread.

## Data Pipeline
ExitInstance acts as a control-flow component in response to a game event. It does not transform data but rather directs the player's state transition.

> **Success Flow:**
> Player Death Event → Server Respawn System → **ExitInstance.respawnPlayer()** → InstancesPlugin.exitInstance() → CommandBuffer schedules player teleport and state change

> **Failure Flow:**
> Player Death Event → Server Respawn System → **ExitInstance.respawnPlayer()** → Exception Thrown → Fallback RespawnController.respawnPlayer() → CommandBuffer schedules standard respawn

