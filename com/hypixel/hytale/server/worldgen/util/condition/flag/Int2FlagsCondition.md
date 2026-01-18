---
description: Architectural reference for Int2FlagsCondition
---

# Int2FlagsCondition

**Package:** com.hypixel.hytale.server.worldgen.util.condition.flag
**Type:** Utility

## Definition
```java
// Signature
@FunctionalInterface
public interface Int2FlagsCondition extends IntUnaryOperator {
```

## Architecture & Concepts
Int2FlagsCondition is a functional interface that defines a behavioral contract for conditional logic within the server-side world generation pipeline. It represents a pure, high-performance function that transforms an integer input, typically representing a set of environmental or contextual flags, into a new integer output, which is also interpreted as a set of flags.

This interface is a core building block for creating composable and efficient world generation rules. By extending Java's IntUnaryOperator, it integrates seamlessly with the Java Streams API and other functional constructs. Its primary purpose is to serve as a predicate or a state transformer in performance-critical loops where object allocation must be minimized. Implementations are expected to be simple bitwise operations, lookups, or mathematical calculations, avoiding any complex or blocking logic.

It acts as a specialized *Strategy Pattern* implementation, allowing different conditional behaviors to be injected into the world generation algorithms without altering their core structure.

## Lifecycle & Ownership
As a functional interface, Int2FlagsCondition does not have a traditional object lifecycle managed by a container or factory. Its lifetime is determined by the scope of its implementation, which is typically a lambda expression or method reference.

- **Creation:** Instances are created ad-hoc by developers when defining world generation rules. They are most commonly instantiated as lambda expressions passed directly as arguments to configuration methods.
- **Scope:** The lifetime of an Int2FlagsCondition instance is bound to the object that holds a reference to it, such as a biome definition or a feature placement rule. It exists for as long as its containing configuration is held in memory.
- **Destruction:** Cleanup is managed entirely by the Java Garbage Collector. When the configuration object holding the lambda is garbage collected, the condition instance is collected with it. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** Implementations of Int2FlagsCondition are **strongly expected to be stateless**. The interface is designed for pure functions where the output depends solely on the input. While it is technically possible for a lambda to capture variables from its enclosing scope (creating a closure with state), this is a dangerous practice in the context of world generation.
- **Thread Safety:** **Not guaranteed; implementation dependent.** A stateless implementation (e.g., one that only performs bitwise logic on its input) is inherently thread-safe. However, if an implementation captures and mutates shared state, it is **not thread-safe** and will cause severe, difficult-to-diagnose concurrency bugs in the multi-threaded world generation engine. All implementations must be written with the assumption that they will be executed concurrently by multiple worker threads.

## API Surface
The public contract consists of a single method to be implemented.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(int var1) | int | O(1) | Evaluates the condition against the input integer flags and returns a new integer representing the resulting flags. Implementations must be non-blocking and computationally inexpensive. |

## Integration Patterns

### Standard Usage
The primary integration pattern is providing a lambda expression to a system that configures world generation rules. The lambda implements the required conditional logic, often using bitmasks.

```java
// A hypothetical rule builder in the world generator
WorldGenRule rule = new WorldGenRule();

// Use a lambda to define a condition: succeed if the 2nd bit is set
// and the 4th bit is not set.
int requiredFlags = 0b0010;
int forbiddenFlags = 0b1000;

rule.setCondition((inputFlags) -> {
    boolean hasRequired = (inputFlags & requiredFlags) == requiredFlags;
    boolean hasForbidden = (inputFlags & forbiddenFlags) != 0;
    return hasRequired && !hasForbidden ? 1 : 0; // Return 1 for success, 0 for failure
});
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Never create a condition that relies on or modifies external mutable state. This will break when executed in parallel by the world generation engine.
    ```java
    // DANGEROUS: Captures and modifies external state
    int counter = 0;
    rule.setCondition((flags) -> {
        counter++; // Unsafe concurrent modification
        return flags & counter;
    });
    ```
- **Blocking Operations:** The eval method must never perform file I/O, network requests, or any other long-running or blocking operation. This will cause catastrophic performance degradation by stalling a world generation thread.

## Data Pipeline
Int2FlagsCondition is a processing node within a larger data flow. It does not manage a data stream itself but acts upon a single data point.

> Flow:
> World Generator State (Integer Flags) -> **Int2FlagsCondition.eval()** -> Result (Integer Flags) -> Conditional Branch in Algorithm

