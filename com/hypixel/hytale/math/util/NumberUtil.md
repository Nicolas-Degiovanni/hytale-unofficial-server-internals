---
description: Architectural reference for NumberUtil
---

# NumberUtil

**Package:** com.hypixel.hytale.math.util
**Type:** Utility

## Definition
```java
// Signature
public class NumberUtil {
```

## Architecture & Concepts
The NumberUtil class provides a collection of static, stateless helper methods for primitive numeric operations. It is a foundational component within the core math library, designed for high-frequency, performance-critical calculations where object allocation and autoboxing must be strictly avoided.

This class is not a service or a managed component; it is a pure utility. Its primary architectural role is to centralize common, low-level mathematical logic, ensuring consistent behavior and providing a single point for future optimization, such as introducing hardware-specific intrinsics. It is expected to be used extensively in game loop calculations, network serialization, and data structure manipulation.

### Lifecycle & Ownership
- **Creation:** As a class with only static members, NumberUtil is never instantiated. Its methods become available upon class loading by the JVM ClassLoader, typically during engine startup.
- **Scope:** The class and its methods are application-scoped. They are available for the entire lifetime of the JVM process.
- **Destruction:** The class is unloaded only when the ClassLoader that loaded it is garbage collected, which for core engine classes, effectively means upon application termination. There is no instance-level destruction or cleanup.

## Internal State & Concurrency
- **State:** NumberUtil is completely stateless. It holds no member variables and its methods operate exclusively on the arguments provided. The output of any method is determined solely by its input.
- **Thread Safety:** This class is inherently thread-safe. Due to its stateless nature, methods can be called concurrently from any thread without risk of race conditions or data corruption. No synchronization primitives such as locks or atomics are required.

## API Surface
The public contract of NumberUtil is minimal and focused on primitive arithmetic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| sum(short a, short b) | short | O(1) | Performs a raw addition of two short primitives. Does not handle overflow. |
| subtract(short a, short b) | short | O(1) | Performs a raw subtraction of two short primitives. Does not handle underflow. |

## Integration Patterns

### Standard Usage
Direct static invocation is the only intended pattern. This avoids any object creation or resolution overhead.

```java
// Correctly use the static methods for calculations.
short currentHealth = 100;
short damage = 25;
short newHealth = NumberUtil.subtract(currentHealth, damage);
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** The class is not designed to be instantiated. Attempting to create an instance is a design violation and will fail if a private constructor is enforced.
- **Redundant Implementation:** Do not re-implement these simple utilities elsewhere. Centralizing them in NumberUtil ensures consistency and maintainability.

## Data Pipeline
NumberUtil does not operate as a standalone pipeline stage. Instead, it serves as a low-level tool *within* a larger data processing flow. Its primary function is to perform discrete, stateless calculations on data as it passes between major architectural components.

> **Example Flow: Player Damage Calculation**
>
> Network Packet -> PacketDeserializer -> GameStateUpdater -> **NumberUtil.subtract()** -> EntityComponentSystem -> ClientRenderer

