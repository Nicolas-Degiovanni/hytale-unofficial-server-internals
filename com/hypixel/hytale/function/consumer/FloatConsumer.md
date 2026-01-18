---
description: Architectural reference for FloatConsumer
---

# FloatConsumer

**Package:** com.hypixel.hytale.function.consumer
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface FloatConsumer {
```

## Architecture & Concepts
FloatConsumer is a functional interface that represents an operation accepting a single primitive float argument and returning no result. This is a core component of the engine's adoption of modern Java functional programming paradigms.

Architecturally, it serves as a specialized, high-performance version of the standard Java Consumer for the primitive float type. By operating directly on the primitive, it avoids the performance overhead associated with autoboxing a float to its wrapper class, Float. This is critical in performance-sensitive systems like the rendering pipeline, physics engine, or animation systems where such operations occur thousands of time per frame.

This interface is the primary mechanism for implementing callbacks, event listeners, and terminal stream operations that involve float values. It promotes a decoupled design, allowing components to provide behavior (the "what") to systems that control execution (the "when").

### Lifecycle & Ownership
- **Creation:** Implementations of FloatConsumer are not instantiated with the new keyword. They are typically provided as lambda expressions or method references at the call site. The Java runtime synthesizes an object that implements the interface.
- **Scope:** The lifetime of a FloatConsumer instance is determined by the object that holds a reference to it. It can be ephemeral, existing only for the duration of a single method call (e.g., a forEach operation on a stream), or it can be long-lived, such as an event listener stored in a manager class until explicitly de-registered.
- **Destruction:** An instance is eligible for garbage collection once all references to it are gone. **Warning:** Long-lived consumers, such as listeners, must be explicitly de-registered from their event sources to prevent memory leaks.

## Internal State & Concurrency
- **State:** The FloatConsumer interface itself is stateless. However, an implementation (particularly a lambda) can be stateful if it captures variables from its enclosing scope. Such "capturing lambdas" can introduce mutable state.
- **Thread Safety:** The interface contract does not enforce thread safety. The safety of any given implementation is the responsibility of its author. If a FloatConsumer implementation accesses or modifies shared mutable state, it must be properly synchronized. **Warning:** Consumers passed to asynchronous or parallel systems are expected to be thread-safe or to confine their operations to thread-safe APIs.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(float value) | void | O(1) | Executes the defined operation on the provided float. The complexity of the underlying implementation is user-defined. |

## Integration Patterns

### Standard Usage
FloatConsumer is used to pass behavior into methods. This is common for event handling or configuring components.

```java
// Example: Using a lambda to set a particle emitter's rate
ParticleSystem system = getParticleSystem();
system.configureRate( (rate) -> {
    // This lambda is an implementation of FloatConsumer
    System.out.println("New particle rate is: " + rate);
});

// Example: Using a method reference for a cleaner syntax
class HealthMonitor {
    public void onHealthChanged(float newHealth) {
        // Logic to update UI or other systems
    }
}

HealthComponent health = player.getHealthComponent();
HealthMonitor monitor = new HealthMonitor();
health.addHealthListener(monitor::onHealthChanged); // Method reference
```

### Anti-Patterns (Do NOT do this)
- **Blocking Implementations:** Never implement a FloatConsumer with a long-running or blocking operation (e.g., file I/O, network requests) if it is being called from a performance-critical thread like the main game loop or rendering thread. This will cause severe frame drops and application unresponsiveness.
- **Stateful Lambdas in Parallel Systems:** Avoid using consumers that modify shared, non-thread-safe state from within a parallel stream or asynchronous task. This is a direct path to race conditions and non-deterministic behavior.

## Data Pipeline
FloatConsumer does not represent a data pipeline itself, but rather a terminal stage or a processing step within one. It is a sink for float values.

> Flow:
> Data Source (e.g., Game Event, UI Slider) -> float value -> **FloatConsumer.accept(value)** -> Side Effect (e.g., State Mutation, Logging, Rendering Call)

