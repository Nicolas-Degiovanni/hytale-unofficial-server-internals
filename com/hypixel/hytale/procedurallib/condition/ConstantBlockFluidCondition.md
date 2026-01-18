---
description: Architectural reference for ConstantBlockFluidCondition
---

# ConstantBlockFluidCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class ConstantBlockFluidCondition implements IBlockFluidCondition {
```

## Architecture & Concepts
The ConstantBlockFluidCondition is a foundational component within the procedural generation framework. It serves as a terminal or leaf node in a condition evaluation tree. Its primary architectural role is to provide a non-negotiable, static boolean outcome—either always true or always false—regardless of the input world data.

This class embodies the Flyweight pattern through its static final instances, DEFAULT_TRUE and DEFAULT_FALSE. By providing these pre-allocated, immutable objects, the system avoids the performance and memory overhead of repeatedly instantiating new condition objects for the most common terminal cases. This is critical in performance-sensitive contexts like chunk generation, where millions of conditions may be evaluated per second.

It acts as a concrete implementation of the Strategy pattern, where the strategy is simply to return a fixed value. This allows algorithms that operate on the IBlockFluidCondition interface to handle constant values without special-casing or null checks.

### Lifecycle & Ownership
- **Creation:** The static instances DEFAULT_TRUE and DEFAULT_FALSE are instantiated by the JVM during class loading and exist for the application's entire lifetime. Custom instances can be created via the public constructor, typically by a higher-level factory or a configuration parser that builds condition graphs from data files.
- **Scope:** The static instances are global and application-scoped. Custom instances are transient and their lifetime is tied to the specific procedural rule or generator that created them.
- **Destruction:** Static instances are reclaimed upon JVM shutdown. Custom instances are subject to standard garbage collection once they are no longer referenced by any part of the procedural generation system.

## Internal State & Concurrency
- **State:** The object's state is immutable. The single field, *result*, is declared final and is set only once during construction. The object holds no other state and performs no caching.
- **Thread Safety:** This class is inherently thread-safe. Due to its immutability and lack of side effects, a single instance can be safely shared and its *eval* method can be called concurrently by any number of threads without synchronization. This makes it ideal for use in parallelized world generation tasks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(int block, int fluid) | boolean | O(1) | Returns the pre-configured boolean result, ignoring all input parameters. |

## Integration Patterns

### Standard Usage
This component is not meant to be used in isolation. It is designed to be plugged into a larger system that evaluates IBlockFluidCondition objects. The static instances should be used whenever possible.

```java
// Correctly use the pre-allocated static instance for a condition
// that must always pass.
IBlockFluidCondition condition = ConstantBlockFluidCondition.DEFAULT_TRUE;

// The condition is then passed to a system that evaluates it.
if (proceduralSystem.checkCondition(x, y, z, condition)) {
    // This block will always execute
}
```

### Anti-Patterns (Do NOT do this)
- **Redundant Instantiation:** Do not create new instances when the static singletons are sufficient. This creates unnecessary garbage collection pressure.

   ```java
   // ANTI-PATTERN: Wasteful object creation
   IBlockFluidCondition badCondition = new ConstantBlockFluidCondition(true);

   // PREFERRED: Zero-allocation access
   IBlockFluidCondition goodCondition = ConstantBlockFluidCondition.DEFAULT_TRUE;
   ```

- **Unnecessary Abstraction:** Do not use this class to wrap a simple boolean in application logic. Its purpose is to satisfy the IBlockFluidCondition interface contract within the procedural library.

## Data Pipeline
This class does not process a pipeline of data. Instead, it acts as a gate or a terminal decision point within a larger data processing flow, such as world generation.

> Flow:
> Procedural Generator -> Evaluates Condition Tree -> **ConstantBlockFluidCondition.eval()** -> Returns Constant Boolean -> Generator Places Block or Skips

