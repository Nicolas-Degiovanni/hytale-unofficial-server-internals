---
description: Architectural reference for DoubleQuadObjectConsumer
---

# DoubleQuadObjectConsumer

**Package:** com.hypixel.hytale.function.consumer
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface DoubleQuadObjectConsumer<T, U, R, V> {
   void accept(double var1, T var3, U var4, R var5, V var6);
}
```

## Architecture & Concepts
DoubleQuadObjectConsumer is a specialized, high-arity functional interface. It serves as a behavioral contract for any operation that consumes five arguments—one primitive double and four generic objects—and returns no result.

Architecturally, this interface is a key component of the engine's functional programming patterns. Its existence is primarily driven by performance optimization. By defining a primitive `double` as the first parameter, it allows engine systems to pass floating-point values without the overhead of autoboxing them into Double objects. This is critical in performance-sensitive contexts such as the rendering pipeline, physics simulation, or particle systems, where millions of such operations may occur per frame.

This interface enables a declarative style, allowing systems to define *what* to do with data, and pass that behavior into methods that control *when* and *how* it is executed. It acts as a callback or a pluggable piece of logic.

### Lifecycle & Ownership
As a functional interface, DoubleQuadObjectConsumer does not have a traditional object lifecycle. Instead, its lifecycle is defined by the lambda expressions or method references that implement it.

- **Creation:** Instances are not created via a constructor. They are implicitly instantiated by the Java runtime when a lambda expression or method reference matching the interface's signature is provided at a call site.
- **Scope:** The scope of an implementation is typically ephemeral and lexical. It exists for the duration of the method call it is passed into. If the lambda captures variables from its enclosing scope (forming a closure), its lifetime may be extended and tied to the captured objects.
- **Destruction:** Instances are managed entirely by the Java Garbage Collector. They become eligible for collection as soon as they are no longer reachable, which is usually after the higher-order function they were passed to completes execution.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, any lambda expression that implements it can be stateful by capturing variables from its enclosing scope. This practice should be approached with extreme caution.

- **Thread Safety:** The interface contract is inherently thread-agnostic. The thread safety of a given implementation is the sole responsibility of the developer providing the lambda. If an implementation modifies shared state from multiple threads without proper synchronization, it will introduce severe concurrency defects.

**WARNING:** Stateful lambdas are a significant source of race conditions when used with parallel streams or asynchronous task executors. Always prefer stateless implementations or ensure that any captured state is immutable or properly synchronized.

## API Surface
The public contract consists of a single abstract method required for implementation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(double, T, U, R, V) | void | O(N) | The functional method. Executes the user-defined logic. Complexity is determined by the implementation. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to pass a lambda expression to a higher-order function that accepts a DoubleQuadObjectConsumer. This is common in systems that iterate over collections of game data and apply a specific operation.

```java
// A hypothetical system that processes entity physics properties
public void forEachPhysicsBody(DoubleQuadObjectConsumer<Entity, Vector3, Quaternion, World> action) {
    for (Entity entity : entities) {
        PhysicsComponent phys = entity.getPhysics();
        // The engine provides the data; the lambda provides the logic
        action.accept(
            engine.getDeltaTime(),
            entity,
            phys.getVelocity(),
            phys.getRotation(),
            entity.getWorld()
        );
    }
}

// Usage:
forEachPhysicsBody((deltaTime, entity, velocity, rotation, world) -> {
    // Apply drag based on delta time
    velocity.multiply(1.0 - (DRAG_COEFFICIENT * deltaTime));
});
```

### Anti-Patterns (Do NOT do this)
- **Stateful Lambdas in Parallel Systems:** Never use a lambda that modifies external, unsynchronized state when passing it to a parallel processing system. This is a guaranteed race condition.
  ```java
  // BAD: This will produce non-deterministic results if run in parallel
  List<Vector3> positions = new ArrayList<>();
  parallelProcessor.execute((dt, entity, vel, rot, world) -> {
      positions.add(entity.getPosition()); // Unsynchronized list modification
  });
  ```

- **Complex Inline Logic:** Avoid embedding large, multi-line, or difficult-to-read logic directly inside a lambda. This harms maintainability. For complex operations, implement the logic in a separate, well-named private method and provide it as a method reference.
  ```java
  // BAD: Hard to read and test
  forEachPhysicsBody((dt, entity, vel, rot, world) -> {
      // 20 lines of complex physics calculations here...
  });

  // GOOD: Clean, readable, and testable
  forEachPhysicsBody(this::applyComplexAerodynamics);
  ```

## Data Pipeline
This interface does not represent a data pipeline itself, but rather a functional stage *within* one. It is a terminal operation (a consumer) that typically results in a side effect, such as mutating game state or dispatching a render command.

> Flow:
> Data Source (e.g., Physics Engine State) -> Iteration Logic (e.g., forEachPhysicsBody) -> **DoubleQuadObjectConsumer Implementation** -> Side Effect (e.g., Entity State Mutation)

