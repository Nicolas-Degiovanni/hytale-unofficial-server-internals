---
description: Architectural reference for TriIntPredicate
---

# TriIntPredicate

**Package:** com.hypixel.hytale.function.predicate
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface TriIntPredicate {
   boolean test(int var1, int var2, int var3);
}
```

## Architecture & Concepts
TriIntPredicate is a specialized functional interface that defines a behavioral contract for a function that accepts three primitive integer arguments and returns a boolean result. It is a core component of the engine's functional programming toolkit, designed for high-performance scenarios where predicate logic is required.

Its primary architectural purpose is to avoid the performance overhead associated with autoboxing and unboxing. A generic equivalent, such as `Predicate<SomeObject>`, would require wrapping primitive integers in `Integer` objects, incurring significant memory allocation and garbage collection pressure in tight loops. By operating directly on `int` primitives, TriIntPredicate is suitable for performance-critical systems like world generation, particle systems, or physics calculations, which frequently operate on three-dimensional integer coordinates (x, y, z).

This interface represents a *strategy* or a *condition* that can be passed as an argument to other methods, enabling flexible and decoupled algorithm design.

### Lifecycle & Ownership
As a functional interface, TriIntPredicate does not have a traditional object lifecycle. Instead, the lifecycle pertains to its *implementations* (typically lambda expressions or method references).

- **Creation:** Implementations are not instantiated via a constructor. They are defined inline as lambda expressions or created as method references at the call site where a TriIntPredicate is expected.
- **Scope:** The lifetime of a TriIntPredicate implementation is bound to the context in which it is defined. A lambda passed to a method is typically stack-scoped and short-lived. If assigned to a class field, its lifetime matches that of the containing object.
- **Destruction:** Implementations are managed by the Java Garbage Collector. They become eligible for collection once they are no longer reachable, such as when the method they were passed to returns or the object holding them is destroyed.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, an implementation (e.g., a lambda) can become stateful if it forms a closure by capturing variables from its enclosing scope. Stateless implementations are strongly preferred.
- **Thread Safety:** The interface contract is inherently thread-safe. The thread safety of a specific implementation is the sole responsibility of the developer providing it.
    - **WARNING:** Stateful implementations that capture and modify non-final or non-thread-safe variables are **not** thread-safe and must not be used in concurrent systems without external synchronization. This is a common source of severe race conditions. Stateless predicates, which rely only on their input arguments, are always thread-safe.

## API Surface
The public contract consists of a single abstract method, defining the functional signature.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(int, int, int) | boolean | Implementation-dependent | Evaluates the predicate against the three integer inputs. Returns true if the condition is met, false otherwise. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to pass a lambda expression to a method that accepts a TriIntPredicate. This allows the method to execute the provided logic without being coupled to the specific condition.

```java
// A hypothetical world utility that finds positions matching a condition.
public class WorldScanner {
    public Set<Vector3i> findPositions(TriIntPredicate condition) {
        Set<Vector3i> results = new HashSet<>();
        for (int x = 0; x < 128; x++) {
            for (int y = 0; y < 128; y++) {
                for (int z = 0; z < 128; z++) {
                    if (condition.test(x, y, z)) {
                        results.add(new Vector3i(x, y, z));
                    }
                }
            }
        }
        return results;
    }
}

// Usage: Find all positions above sea level (e.g., y > 64)
WorldScanner scanner = new WorldScanner();
Set<Vector3i> surfacePositions = scanner.findPositions((x, y, z) -> y > 64);
```

### Anti-Patterns (Do NOT do this)
- **Complex Inline Logic:** Avoid writing multi-line, complex business logic directly inside a lambda. This harms readability and prevents reuse. For complex conditions, define a private method with a descriptive name and pass it as a method reference.
    ```java
    // BAD: Unreadable lambda
    scanner.findPositions((x, y, z) -> (x * x + z * z < 1000) && (y > 64 && y < 80) && (Math.abs(x-z) % 2 == 0));

    // GOOD: Clear, reusable method reference
    private boolean isCaveEntrance(int x, int y, int z) {
        // ... complex logic here ...
    }
    Set<Vector3i> entrances = scanner.findPositions(this::isCaveEntrance);
    ```
- **Stateful Lambdas in Concurrent Code:** Never use a lambda that modifies captured state from a concurrent context without proper synchronization. This is extremely dangerous.

## Data Pipeline
TriIntPredicate does not process a data stream itself; rather, it acts as a conditional gate or filter *within* a larger data pipeline. It is a component that makes a decision based on incoming data points.

> **Flow:**
> Data Source (e.g., Voxel Iterator) -> **TriIntPredicate (Filter Logic)** -> Downstream Processor (e.g., Block Placer, Renderer)

