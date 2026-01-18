---
description: Architectural reference for EntityLodCommand
---

# EntityLodCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity
**Type:** Transient

## Definition
```java
// Signature
public class EntityLodCommand extends CommandBase {
```

## Architecture & Concepts
The EntityLodCommand class is a concrete implementation of the Command Pattern, designed to be discovered and managed by the server's central command system. Its primary architectural function is to expose a low-level entity culling parameter to server administrators via a simple chat command.

This command acts as a direct, high-level interface to a global, static configuration value within the entity tracking system: `LegacyEntityTrackerSystems.LegacyLODCull.ENTITY_LOD_RATIO`. This design choice creates a tight coupling between the command layer and the entity culling logic. While this provides a powerful and immediate mechanism for live performance tuning, it bypasses more robust configuration services. This suggests its intended use is for debugging or emergency administrative action rather than standard server configuration.

The class also demonstrates the composite command pattern by including a nested static class, `Default`, which functions as a subcommand to reset the value to a hardcoded default.

## Lifecycle & Ownership
- **Creation:** A single instance of EntityLodCommand is created by the command registration system during server bootstrap. The system scans for classes extending CommandBase and instantiates them.
- **Scope:** The object instance is long-lived, persisting for the entire server session. It is held in a central registry, typically a Map, within the core command service.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** The EntityLodCommand instance itself is effectively stateless. It holds a final reference to its argument definition (`ratioArg`), but this state does not change after construction.
- **Thread Safety:** This class is **not thread-safe**. Its `executeSync` method directly mutates a global static variable (`ENTITY_LOD_RATIO`). The name `executeSync` strongly implies that the command system guarantees its execution on the main server thread, which is the sole mechanism preventing race conditions.

**WARNING:** Any external system that modifies `LegacyEntityTrackerSystems.LegacyLODCull.ENTITY_LOD_RATIO` from another thread will introduce severe concurrency bugs. This variable must only be accessed from the main server tick thread.

## API Surface
The public contract is defined by its constructor and the `executeSync` method, which is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EntityLodCommand() | constructor | O(1) | Registers the command name `lod`, its description, and its required `ratio` argument. It also registers the `default` subcommand. |
| executeSync(context) | void | O(1) | Parses the `ratio` argument from the context and performs a direct write to the global static `ENTITY_LOD_RATIO` field. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is invoked exclusively by the server's command dispatcher in response to player or console input.

A server administrator would use this command via the chat or console:
```
// Sets a custom entity LOD ratio
/lod 0.00005

// Resets the ratio to its hardcoded default
/lod default
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EntityLodCommand()`. The command system is responsible for the lifecycle of all command objects. Direct instantiation will result in a non-registered, orphaned command.
- **Manual Invocation:** Avoid calling `executeSync` directly. Doing so bypasses the command system's critical infrastructure, including permission checks, argument parsing, and exception handling.
- **External State Mutation:** Do not modify `LegacyEntityTrackerSystems.LegacyLODCull.ENTITY_LOD_RATIO` from other systems without extreme care. This command assumes it has authoritative control over the value during its execution.

## Data Pipeline
The flow of data for this command begins with user input and ends with a modification of a global server state variable.

> Flow:
> Player Input (`/lod 0.00005`) -> Network Packet -> Command Dispatcher -> Argument Parser -> **EntityLodCommand.executeSync** -> Direct static field write to `ENTITY_LOD_RATIO` -> Confirmation Message -> Network Layer -> Player Chat UI

