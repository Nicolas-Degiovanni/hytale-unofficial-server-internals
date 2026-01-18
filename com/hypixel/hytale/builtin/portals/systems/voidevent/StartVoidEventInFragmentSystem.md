---
description: Architectural reference for StartVoidEventInFragmentSystem
---

# StartVoidEventInFragmentSystem

**Package:** com.hypixel.hytale.builtin.portals.systems.voidevent
**Type:** Transient

## Definition
```java
// Signature
public class StartVoidEventInFragmentSystem extends DelayedSystem<EntityStore> {
```

## Architecture & Concepts
The StartVoidEventInFragmentSystem is a time-based controller responsible for managing the lifecycle of a Void Event within a specific game world fragment. It operates within the server-side Entity Component System (ECS) framework.

Architecturally, this system functions as a state machine trigger. It does not contain the logic for the Void Event itself. Instead, its sole responsibility is to monitor the passage of time and world state, and based on pre-defined conditions, either create or destroy the primary **VoidEvent** entity.

This class extends DelayedSystem, a crucial architectural choice indicating it does not execute on every game tick. The constructor argument of 1.0F configures it to run approximately once per second, making it an efficient, low-overhead monitor for a long-running game event. It reads its configuration from the PortalWorld resource, decoupling the event's trigger logic from the system's implementation.

## Lifecycle & Ownership
- **Creation:** An instance of StartVoidEventInFragmentSystem is created by the engine's System Manager when a world is loaded that has this system registered for its context. It is not created manually by developers.
- **Scope:** The object's lifetime is bound to the world or game fragment it is managing. It is not a global or session-wide singleton.
- **Destruction:** The instance is destroyed and garbage collected when the associated world or fragment is unloaded from memory.

## Internal State & Concurrency
- **State:** This class is **stateless**. It holds no mutable member fields and derives all necessary information from the Store and its associated Resources (PortalWorld) during each execution of delayedTick. This is a common and desirable pattern for ECS systems, as it prevents state corruption and simplifies execution flow.
- **Thread Safety:** This system is **not thread-safe**. It is designed to be executed exclusively by the main server world thread as part of the engine's scheduled system updates. All interactions with the EntityStore and its resources must be synchronized through the engine's game loop. Unmanaged, multi-threaded access will lead to world corruption.

## API Surface
The public API is minimal, intended for invocation by the engine's System Manager, not by user code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| delayedTick(dt, systemIndex, store) | void | O(1) | The primary entry point. Evaluates timing conditions and may issue a command to the EntityStore to create or destroy the VoidEvent entity. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. Its behavior is configured entirely through the data it consumes from the PortalWorld resource. To control when a Void Event starts, a developer would modify the VoidEventConfig associated with a given PortalWorld. The system will automatically detect and act upon these data changes during its next scheduled tick.

```java
// This system is not called directly.
// The following is a conceptual example of how its data source is configured.

// Somewhere in world setup code...
PortalWorld portalWorld = store.getResource(PortalWorld.getResourceType());
VoidEventConfig config = portalWorld.getVoidEventConfig();

// By setting this value, you are configuring the behavior of
// the StartVoidEventInFragmentSystem.
config.setShouldStartAfterSeconds(300); // Event will now start after 5 minutes.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new StartVoidEventInFragmentSystem()`. It will not be registered with the engine's scheduler and will have no effect.
- **Manual Invocation:** Do not call the delayedTick method manually. This bypasses the engine's timing mechanism and can lead to race conditions, duplicate event entities, or other unpredictable world state errors.

## Data Pipeline
This system acts as a simple conditional processor in a larger data flow. It translates time-based world state into entity lifecycle commands.

> Flow:
> PortalWorld Resource (Configuration) -> **StartVoidEventInFragmentSystem** (Time & Condition Check) -> EntityStore Command (Add/Remove Entity) -> World State Change

