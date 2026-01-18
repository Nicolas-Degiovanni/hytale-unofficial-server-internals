---
description: Architectural reference for TriIntObjectDoubleToByteFunction
---

# TriIntObjectDoubleToByteFunction<T>

**Package:** com.hypixel.hytale.function.function
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface TriIntObjectDoubleToByteFunction<T> {
```

## Architecture & Concepts
TriIntObjectDoubleToByteFunction is a specialized functional interface that defines a contract for a function. It is a core component of the engine's data processing framework, designed for high-performance, low-allocation scenarios. Its primary role is to encapsulate a single, stateless operation that can be passed as a parameter to other systems.

This pattern is heavily utilized in performance-critical systems like world generation, lighting propagation, and bulk data manipulation. The name signature strongly implies its use case: operating on a data point defined by three integer coordinates (e.g., X, Y, Z), a generic context object (T), and a double-precision input value, ultimately producing a single byte result. This is a classic pattern for voxel or grid-based calculations where the function represents the "kernel" of a larger processing job.

By using a functional interface instead of a traditional class-based strategy pattern, the engine can leverage lambda expressions and method references, significantly reducing boilerplate code and avoiding heap allocations for single-use operation objects.

## Lifecycle & Ownership
As an interface, TriIntObjectDoubleToByteFunction has no lifecycle itself. The lifecycle described here pertains to the *implementing instance*, which is typically a lambda expression or a method reference.

- **Creation:** An instance is created implicitly when a lambda expression is assigned or passed as an argument to a method. For example, `(x, y, z, context, value) -> (byte)0` creates a new function object.
- **Scope:** The lifetime of the function instance is tied to the object that holds a reference to it. If passed as a method argument, it may be ephemeral and eligible for garbage collection after the method returns. If stored in a field of a long-lived service, it will persist for the lifetime of that service.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup requirements.

## Internal State & Concurrency
- **State:** The interface contract is inherently stateless. However, implementations created via lambda expressions can become stateful by capturing variables from their enclosing scope (forming a closure). This practice is strongly discouraged in concurrent contexts.

- **Thread Safety:** The interface itself is thread-agnostic. The thread safety of a given implementation is the sole responsibility of the developer who writes it.

    **WARNING:** Implementations that capture and mutate external state are **not thread-safe** by default. If a system might execute these functions in parallel (e.g., a multi-threaded chunk generator), any captured variables must be immutable or access to them must be synchronized, which would defeat the performance goals of this interface.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(int, int, int, T, double) | byte | O(impl) | Executes the function logic. The complexity is entirely dependent on the specific implementation provided. |

## Integration Patterns

### Standard Usage
This interface is not meant to be implemented by a standalone, named class. It should be used inline to provide behavior to a system that processes grid data.

```java
// A system like a WorldGenerator would accept this function to define a rule.
WorldGenerator generator = context.getService(WorldGenerator.class);
Region targetRegion = ...;
WorldGenerationContext worldCtx = ...;

// Define a function that sets blocks to stone if a noise value is > 0.5
TriIntObjectDoubleToByteFunction<WorldGenerationContext> rule =
    (x, y, z, ctx, noiseValue) -> {
        return noiseValue > 0.5 ? Block.STONE_ID : Block.AIR_ID;
    };

// The generator executes this rule for every block in the region.
generator.applyRule(targetRegion, worldCtx, rule);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not write lambdas that modify state outside their own scope. This creates non-local side effects that are difficult to debug and paralyze parallel execution.

    ```java
    // BAD: This lambda modifies a shared counter, creating a race condition
    // if executed by multiple threads.
    AtomicInteger blockCounter = new AtomicInteger(0);
    TriIntObjectDoubleToByteFunction<WorldContext> badRule =
        (x, y, z, ctx, val) -> {
            blockCounter.incrementAndGet(); // Side effect!
            return Block.DIRT_ID;
        };
    ```

- **Complex Logic:** The implementation of this function should be trivial and focused. If your logic involves multiple steps, branching, or calls to other services, encapsulate it in a proper class and provide a method reference if appropriate.

## Data Pipeline
The component acts as a pure transformation function. It does not fetch or push data; it is merely a conduit for logic.

> Flow:
> (int, int, int, T, double) Arguments -> **[Implementation Logic]** -> byte Result

