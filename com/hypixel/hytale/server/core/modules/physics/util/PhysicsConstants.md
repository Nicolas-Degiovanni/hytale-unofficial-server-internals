---
description: Architectural reference for PhysicsConstants
---

# PhysicsConstants

**Package:** com.hypixel.hytale.server.core.modules.physics.util
**Type:** Utility

## Definition
```java
// Signature
public class PhysicsConstants {
```

## Architecture & Concepts
The PhysicsConstants class serves as a centralized, authoritative source for fundamental physical constants used throughout the server-side physics engine. Its primary design goal is to eliminate "magic numbers" from physics calculations, ensuring that all components (e.g., entity movement, projectile trajectory, environmental effects) operate under a consistent and easily tunable physical model.

This class is a foundational, low-level utility. It is not a service and holds no state. It is intended to be statically accessed by any system requiring canonical values for physical simulation.

## Lifecycle & Ownership
As a utility class composed exclusively of static final members, PhysicsConstants does not follow a traditional object lifecycle.

- **Creation:** The class is loaded and initialized by the Java Virtual Machine's ClassLoader when it is first referenced by another component. There is no explicit instantiation.
- **Scope:** Its scope is static. The constants are available for the entire lifetime of the server process once the class has been loaded.
- **Destruction:** The class and its static members are unloaded only when the JVM shuts down. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** This class is stateless. It contains only compile-time constants (static final fields), which are immutable after class initialization.
- **Thread Safety:** This class is inherently thread-safe. Its constants can be safely accessed from any thread without synchronization, as they are immutable and globally accessible.

## API Surface
The public contract consists of globally accessible constants.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| GRAVITY_ACCELERATION | public static final double | O(1) | The standard downward acceleration applied to physics-enabled entities. Units are in blocks per second squared. |

## Integration Patterns

### Standard Usage
Constants should be accessed statically from any class performing physics calculations. There is no need to acquire an instance or reference.

```java
// Example of applying gravity in an entity update tick
double verticalVelocityChange = PhysicsConstants.GRAVITY_ACCELERATION * deltaTime;
entity.getVelocity().subtract(0, verticalVelocityChange, 0);
```

### Anti-Patterns (Do NOT do this)
- **Local Definitions:** Do not redefine these constants locally or in other classes. This defeats the purpose of centralization and creates a high risk of inconsistent physics behavior.
- **Instantiation:** The class is not designed to be instantiated. Attempting to call `new PhysicsConstants()` will fail as there is no public constructor.

## Data Pipeline
PhysicsConstants does not participate in a data pipeline as a processing stage. Instead, it acts as a static **Data Source** for configuration values.

> Flow:
> **PhysicsConstants** -> Physics Engine -> Entity State Update

