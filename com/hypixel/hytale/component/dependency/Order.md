---
description: Architectural reference for Order
---

# Order

**Package:** com.hypixel.hytale.component.dependency
**Type:** Utility

## Definition
```java
// Signature
public enum Order {
```

## Architecture & Concepts
The Order enum is a fundamental construct within the engine's component dependency and initialization framework. It provides a type-safe, explicit mechanism for defining the relative loading and update sequence between two or more components.

This enum is not a standalone service but rather a critical configuration value used by the Dependency Resolver. Its primary function is to establish directed edges in the component dependency graph. By specifying BEFORE or AFTER, developers dictate the topological sort of the component lifecycle, ensuring a deterministic and predictable startup and shutdown sequence. This system is crucial for preventing race conditions and ensuring that components are available before their dependents attempt to access them.

Use of this enum is typically found within annotations, such as a hypothetical *DependsOn* or *UpdateOrder* annotation, which are then processed by the engine's bootstrap coordinator.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine (JVM) when the Order class is first loaded by the ClassLoader. This process is guaranteed to happen only once.
- **Scope:** The instances BEFORE and AFTER are static, final, and persist for the entire lifetime of the application. They are effectively global, immutable singletons.
- **Destruction:** The enum instances are garbage collected when the application's ClassLoader is unloaded, which typically occurs only during JVM shutdown.

## Internal State & Concurrency
- **State:** The Order enum is stateless and immutable. Its constants, BEFORE and AFTER, have no mutable fields and their identity is fixed at compile time.
- **Thread Safety:** This enum is inherently thread-safe. The JVM guarantees safe initialization, and its immutable nature allows it to be read from any thread without requiring synchronization primitives. It is a canonical example of a thread-safe type.

## API Surface
The public contract consists solely of its predefined constants.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BEFORE | Order | O(1) | A constant specifying that a component should be processed before its dependency. |
| AFTER | Order | O(1) | A constant specifying that a component should be processed after its dependency. |

## Integration Patterns

### Standard Usage
The Order enum should only be used as a value within configurations or annotations that control component lifecycle ordering. It serves as a declarative instruction to the dependency management system.

```java
// Hypothetical annotation usage to control initialization order
@ProcessAfter(PhysicsSystem.class) // This component will initialize AFTER PhysicsSystem
public class CharacterControllerSystem extends GameSystem {
    // ...
}

// The dependency resolver would interpret this as a relationship
// where the Order is implicitly AFTER.
```

### Anti-Patterns (Do NOT do this)
- **Ordinal-Based Logic:** Do not use the ordinal() method to derive logic. The order of declaration (BEFORE=0, AFTER=1) is an implementation detail and is extremely brittle. Future additions to the enum could break any logic that relies on these integer values. Always use direct comparison.
- **Serialization of Ordinals:** Persisting the ordinal value is a dangerous practice. If the enum declaration order changes in a future version, saved data or network packets will be deserialized incorrectly. If serialization is required, persist the string name via the name() method.

## Data Pipeline
The Order enum does not process data itself. Instead, it acts as a static routing instruction that directs the flow of the component initialization pipeline.

> Flow:
> Component Annotation Scanning -> Dependency Resolver -> **Order Enum Value** -> Topological Sort Algorithm -> Ordered Component List -> Component Initializer

