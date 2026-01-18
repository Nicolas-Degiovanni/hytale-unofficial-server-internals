---
description: Architectural reference for ObjectiveLocationTriggerCondition
---

# ObjectiveLocationTriggerCondition

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.triggercondition
**Type:** Polymorphic Base Class

## Definition
```java
// Signature
public abstract class ObjectiveLocationTriggerCondition {
```

## Architecture & Concepts
The ObjectiveLocationTriggerCondition class is an abstract base that defines the contract for all trigger conditions related to an objective's physical location in the world. It forms the core of a data-driven, polymorphic system responsible for evaluating world state against predefined criteria within Hytale's Adventure Mode objective framework.

Its primary architectural significance lies in the static CODEC field, a CodecMapCodec. This mechanism acts as a factory and registry, mapping string identifiers from configuration files (e.g., "HourRange", "Weather") to concrete Java class implementations. This design decouples the objective evaluation logic from the specific conditions, allowing designers and modders to define complex objective triggers in data without modifying core engine code.

This class is not used directly but serves as the parent for specific implementations like HourRangeTriggerCondition and WeatherTriggerCondition. The engine's objective system maintains collections of these objects, evaluating them periodically to determine if an objective's state should change.

### Lifecycle & Ownership
- **Creation:** Instances are not created via a constructor. They are deserialized from game configuration data by the central CODEC system. The CodecMapCodec reads a "Type" field from the data source and instantiates the corresponding registered subclass.
- **Scope:** The lifecycle of an ObjectiveLocationTriggerCondition instance is bound to the lifecycle of its parent objective configuration. It is loaded when the adventure or quest is initialized and persists in memory as an immutable configuration object.
- **Destruction:** The object is marked for garbage collection when the parent objective or adventure configuration is unloaded from memory, for example, when a player leaves a zone or completes a quest line.

## Internal State & Concurrency
- **State:** Instances of this class and its subclasses are effectively immutable after deserialization. Their internal fields represent static configuration (e.g., a required weather type) and do not change during runtime. The core evaluation logic is stateless.
- **Thread Safety:** The class itself is inherently thread-safe due to its immutable nature. The primary method, isConditionMet, operates on world state passed in as arguments. Concurrency concerns are therefore deferred to the caller; the engine ensures that calls to this method are made from the main server thread during the game tick to prevent race conditions when accessing world state like EntityStore.

## API Surface
The public contract is minimal, focusing entirely on evaluation and serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isConditionMet(...) | boolean | O(1) | Abstract method. Evaluates if the condition is met for a given location and world state. Subclasses provide the concrete logic. |
| CODEC | CodecMapCodec | N/A | **Critical.** The static factory and registry for all condition types. Used by the engine to deserialize conditions from data. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is rare. The primary pattern is to define a new condition type and register it with the central CODEC. The engine's objective system handles the evaluation loop.

A developer extending the system would create a new subclass and register it.

```java
// 1. Define a new condition class
public class MyCustomTriggerCondition extends ObjectiveLocationTriggerCondition {
    // ... implementation of isConditionMet
}

// 2. Register it with the central CODEC during system initialization
// This is typically done in a static block or bootstrap method.
ObjectiveLocationTriggerCondition.CODEC.register(
    "MyCustomCondition",
    MyCustomTriggerCondition.class,
    MyCustomTriggerCondition.CODEC
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new HourRangeTriggerCondition()`. The system is designed to be data-driven. All instances must be created via the CODEC deserialization pipeline to ensure they are correctly configured.
- **Stateful Implementations:** Subclasses should not contain mutable state. A condition must return the same result for the same world state input. Storing state between calls can lead to severe and difficult-to-debug logical errors in objective progression.
- **External Registration:** Do not attempt to register new condition types after the server has finished its initial bootstrap phase. The CODEC is not designed for dynamic, runtime registration and doing so may lead to race conditions or inconsistent behavior across clients and servers.

## Data Pipeline
This component sits at the intersection of game configuration and live game state evaluation.

> Flow:
> Adventure Mode JSON/Config File -> Hytale Codec System -> **ObjectiveLocationTriggerCondition.CODEC** (Deserialization) -> In-memory Objective Configuration -> Objective System Evaluator -> isConditionMet(currentWorldState) -> Objective State Update

