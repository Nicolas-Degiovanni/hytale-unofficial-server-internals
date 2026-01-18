---
description: Architectural reference for BuilderActionTimerStop
---

# BuilderActionTimerStop

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionTimerStop extends BuilderActionTimer {
```

## Architecture & Concepts
The BuilderActionTimerStop class is a component builder within the server-side NPC behavior system. It embodies a specific, declarative action: stopping a named timer. This class is a concrete implementation of the Builder pattern, designed to be instantiated by the asset loading system, not by direct developer invocation.

Its primary architectural role is to act as a bridge between static data assets (e.g., JSON files defining an NPC's behavior tree) and live, in-game components. When an NPC asset specifies an action to "stop a timer", the asset deserializer instantiates this builder. The NPC's initialization logic then invokes the **build** method on this object to create an executable ActionTimer component, which is subsequently attached to the NPC entity.

This pattern decouples the definition of an NPC's logic from its runtime implementation, allowing designers to configure complex behaviors without writing code. BuilderActionTimerStop is a simple, single-purpose factory responsible for producing one specific type of ActionTimer.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's asset deserialization pipeline when processing an NPC behavior asset. It is never created manually.
- **Scope:** Extremely short-lived. An instance of this class exists only for the brief moment between asset parsing and the final construction of the runtime ActionTimer component.
- **Destruction:** The object is immediately eligible for garbage collection after the **build** method has been called. It holds no references and is not retained by any system.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields and its behavior is fixed. The configuration for the action it builds is hardcoded within its method implementations.
- **Thread Safety:** This class is inherently **thread-safe**. As a stateless object, it can be safely used across multiple threads without locks or synchronization primitives. However, in practice, it is typically confined to the main server thread during the asset loading phase.

## API Surface
The public API is minimal, focused entirely on its role as a factory and descriptor for the asset system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionTimer | O(1) | Constructs and returns a new ActionTimer component configured with the STOP action. |
| getShortDescription() | String | O(1) | Provides a human-readable summary for tooling and debugging. |
| getLongDescription() | String | O(1) | Provides a detailed human-readable description for tooling. |
| getTimerAction() | Timer.TimerAction | O(1) | Returns the hardcoded Timer.TimerAction.STOP enum constant. This is the core logic payload. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in gameplay logic code. It is used declaratively within NPC asset files. The system interacts with it as follows during the NPC initialization phase.

```java
// Conceptual example of how the NPC system uses the builder
// This code is internal to the engine; do not replicate.

// 1. Asset loader deserializes an asset into a builder instance.
BuilderActionTimerStop builder = assetLoader.load("npc_behavior.json");

// 2. The NPC factory invokes build() to create the runtime component.
ActionTimer stopAction = builder.build(npc.getBuilderSupport());

// 3. The component is added to the NPC's behavior system.
npc.getComponentRegistry().add(stopAction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call **new BuilderActionTimerStop()**. The asset system is solely responsible for the lifecycle of builder objects. Manual creation bypasses the data-driven design and will likely fail or lead to unmanaged components.
- **State Modification:** Do not attempt to extend this class to add state. Builders in this system are designed to be stateless translators of asset data.

## Data Pipeline
The BuilderActionTimerStop serves as a critical translation step in the data pipeline that transforms a static asset into a functioning server-side component.

> Flow:
> NPC Behavior Asset (JSON) -> Asset Deserializer -> **BuilderActionTimerStop Instance** -> build() Invocation -> ActionTimer Component -> NPC Behavior System

