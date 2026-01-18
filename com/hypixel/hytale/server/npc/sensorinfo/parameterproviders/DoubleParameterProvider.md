---
description: Architectural reference for DoubleParameterProvider
---

# DoubleParameterProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo.parameterproviders
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface DoubleParameterProvider extends ParameterProvider {
```

## Architecture & Concepts
The DoubleParameterProvider interface defines a strict contract for components that supply a single, double-precision floating-point value to the NPC AI system. It is a fundamental abstraction within the AI Sensor framework, designed to decouple AI decision-making logic from the concrete sources of world-state information.

In essence, this interface answers the question: "How does an AI sensor get a numerical value about the world?" For example, a sensor might need to know the distance to a target, the current health of an entity, or the time elapsed since a specific event. By depending on this interface, the sensor's logic remains generic and reusable, while the specific implementation (e.g., DistanceToTargetProvider, EntityHealthProvider) can be swapped out as needed.

This pattern is critical for building modular and extensible AI behaviors. It allows designers and engineers to create new data sources for AI without modifying the core sensor or behavior tree logic.

## Lifecycle & Ownership
As an interface, DoubleParameterProvider does not have a lifecycle of its own. The lifecycle and ownership semantics are entirely dictated by the concrete class that implements it.

- **Creation:** An implementing class is typically instantiated and configured alongside the AI Sensor that consumes it. This is often done during the NPC's initialization phase.
- **Scope:** The provider's lifetime is tied to its owning Sensor or AI component. It generally persists as long as the parent component is active.
- **Destruction:** The provider is eligible for garbage collection when its owning Sensor is destroyed. No explicit cleanup is defined by this contract.

**WARNING:** Implementers must not assume a specific lifecycle. The provider could be created and destroyed frequently or persist for the entire life of an NPC.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, concrete implementations may be stateful. For instance, a provider might cache a calculated value for the duration of a single game tick to avoid redundant computations.
- **Thread Safety:** This contract provides no guarantees of thread safety. It is the sole responsibility of the implementing class to ensure safe concurrent access if required. In standard practice, providers are accessed exclusively from the main server thread during the AI update cycle.

**WARNING:** Do not assume single-threaded access. While typical, engine-level changes or custom AI systems could introduce multi-threaded scenarios. Implementations that manage mutable state must use appropriate synchronization mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDoubleParameter() | double | *Varies* | Retrieves the parameter value. The complexity is entirely dependent on the implementation. |
| NOT_PROVIDED | double | O(1) | A static sentinel value used to indicate that the parameter is not available or could not be computed. |

## Integration Patterns

### Standard Usage
A concrete class implements the interface to provide a specific piece of data. This provider is then passed to an AI Sensor or Behavior, which calls getDoubleParameter to retrieve the value for its logic.

```java
// Example of a concrete implementation
public class DistanceToPlayerProvider implements DoubleParameterProvider {
    private final Entity self;
    private final Player target;

    // Constructor...

    @Override
    public double getDoubleParameter() {
        if (target == null || !target.isOnline()) {
            return DoubleParameterProvider.NOT_PROVIDED;
        }
        return self.getPosition().distanceTo(target.getPosition());
    }
}

// Usage within an AI component
DoubleParameterProvider distanceProvider = new DistanceToPlayerProvider(npc, player);
double distance = distanceProvider.getDoubleParameter();

if (distance != DoubleParameterProvider.NOT_PROVIDED && distance < 10.0) {
    // Execute close-range behavior
}
```

### Anti-Patterns (Do NOT do this)
- **Returning Magic Numbers:** Do not return arbitrary values like -1 or 0 to indicate an absence of data. Always use the `NOT_PROVIDED` sentinel value. AI logic is built to explicitly check for this constant.
- **Ignoring The Sentinel Value:** Consumers of this interface must *always* check if the returned value is `NOT_PROVIDED` before using it in calculations. Failure to do so can lead to severe AI logic errors.
- **Heavy Computation:** Avoid performing computationally expensive operations (e.g., complex pathfinding, broad world queries) directly within getDoubleParameter. This method may be called every tick, and poor performance will degrade the entire server. Cache results where possible.
- **Throwing Exceptions:** The contract expects a double value, not an exception. Handle internal errors gracefully and return `NOT_PROVIDED` to signal failure.

## Data Pipeline
This interface acts as a data source, initiating a flow of information into the AI system.

> Flow:
> Raw Game State (e.g., Entity Positions, Health) -> **Concrete DoubleParameterProvider Implementation** -> AI Sensor Logic -> AI Behavior Tree Node

