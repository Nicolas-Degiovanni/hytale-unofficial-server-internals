---
description: Architectural reference for IntParameterProvider
---

# IntParameterProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo.parameterproviders
**Type:** Interface

## Definition
```java
// Signature
public interface IntParameterProvider extends ParameterProvider {
```

## Architecture & Concepts
The IntParameterProvider interface defines a strict contract for components that supply a single integer value to the NPC AI system. It is a core component of the AI Sensor framework, abstracting the *source* of an integer parameter from its *consumer*. This follows the Strategy Pattern, allowing AI behaviors to be configured with various data sources without altering the sensor's logic.

For example, a sensor that measures the "threat level" of a target can be configured with different providers: one that reads a static value from a configuration file, another that calculates it based on the target's equipment, and a third that queries a dynamic world state manager. The sensor itself only interacts with the IntParameterProvider contract and remains agnostic to the underlying data source.

This abstraction is critical for creating flexible, data-driven, and reusable AI components.

### Lifecycle & Ownership
- **Creation:** As an interface, IntParameterProvider is not instantiated directly. Concrete implementations are instantiated by the AI Behavior Loading system when parsing NPC definition files or by other systems that dynamically construct AI logic.
- **Scope:** The lifecycle of a provider instance is tied to the lifecycle of the AI component (e.g., a Sensor or Behavior) that owns it. It typically exists for as long as the parent NPC entity is active.
- **Destruction:** The provider is eligible for garbage collection when its owning AI component is destroyed, usually as part of the NPC's despawn or death sequence.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, concrete implementations may be stateful or stateless. A stateful provider might cache a calculated value for a single game tick, while a stateless provider might read directly from an entity's component state on every call.
- **Thread Safety:** This interface provides no intrinsic thread safety guarantees. Concurrency control is the responsibility of the concrete implementation. Consumers, such as AI sensors operating within the main server thread, should assume that providers are **not thread-safe** and must only be accessed from the owning entity's update tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NOT_PROVIDED | static int | O(1) | A sentinel value (Integer.MIN_VALUE) indicating that the provider cannot supply a valid integer at this time. This is not an error state but a valid, expected signal for "no data". |
| getIntParameter() | int | Implementation-dependent | Retrieves the integer parameter. Callers MUST check the return value against NOT_PROVIDED to handle cases where data is unavailable. |

## Integration Patterns

### Standard Usage
A consumer, such as an AI sensor, must always check for the sentinel value before using the result. This defensive check is mandatory for stable AI behavior.

```java
// Correctly consuming an IntParameterProvider
void processThreatSensor(IntParameterProvider threatProvider) {
    int threatLevel = threatProvider.getIntParameter();

    if (threatLevel == IntParameterProvider.NOT_PROVIDED) {
        // Handle the case where no threat level is available
        // e.g., default to a neutral behavior
        return;
    }

    // Proceed with AI logic using the valid threatLevel
    executeBehavior(threatLevel);
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Sentinel Value:** Failing to check for NOT_PROVIDED will lead to critical logic errors, as Integer.MIN_VALUE will be processed as a real parameter, causing unpredictable AI decisions.
- **Assuming a Range:** Do not assume the returned integer will be positive or within any specific range unless the concrete implementation guarantees it. The contract only ensures an integer or the sentinel value.
- **Direct Implementation Coupling:** Avoid casting the provider to a specific concrete class. Code should be written against the interface to maintain flexibility.

## Data Pipeline
The data flow for this component is typically from a source of game state, through a concrete provider, and into an AI decision-making system.

> Flow:
> World State / Entity Component -> **Concrete IntParameterProvider** -> Sensor Logic -> Behavior Tree Node -> NPC Action

