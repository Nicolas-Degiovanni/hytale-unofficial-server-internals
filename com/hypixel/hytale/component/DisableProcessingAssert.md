---
description: Architectural reference for DisableProcessingAssert
---

# DisableProcessingAssert

**Package:** com.hypixel.hytale.component
**Type:** Marker Interface

## Definition
```java
// Signature
@Deprecated(forRemoval = true)
public interface DisableProcessingAssert {
}
```

## Architecture & Concepts
DisableProcessingAssert is a marker interface used to signal to the core component processing system that a specific component should be exempt from certain runtime assertions. Specifically, it indicates that a component is intentionally designed to not be processed by a standard system during the game loop's update tick.

This interface acts as a contract or a tag. Systems that iterate over components can perform an *instanceof* check for DisableProcessingAssert. If the check returns true, the system can safely skip processing the component without logging a warning or throwing an error, which would normally occur for an unhandled component type.

**WARNING:** This interface is deprecated and marked for removal. Its use was a temporary solution to handle special-case components that did not fit the standard entity-component-system (ECS) processing model. Modern architectural patterns within the engine have superseded this mechanism, typically by using more explicit registration or configuration-based approaches to manage component processing lifecycles.

## Lifecycle & Ownership
- **Creation:** As an interface, DisableProcessingAssert is not instantiated. It is *implemented* by concrete component classes. The lifecycle is therefore tied to the component that implements it.
- **Scope:** The scope of this marker is compile-time and runtime type-checking. It exists for as long as the implementing class is loaded by the JVM.
- **Destruction:** Not applicable. The interface itself is a static definition.

## Internal State & Concurrency
- **State:** This interface is stateless. It contains no fields and defines no methods, serving only as a type marker.
- **Thread Safety:** Not applicable. Thread safety is the responsibility of the class that implements this interface. The interface itself introduces no concurrency concerns.

## API Surface
This interface exposes no public API. It is a pure marker interface and has no methods or fields.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| (none) | - | - | This interface has no members. |

## Integration Patterns

### Standard Usage
**WARNING:** The following pattern is obsolete and should not be used in new code. It is documented for maintenance of legacy systems only.

A component would implement this interface to signal its special-cased nature to the engine.

```java
// Legacy component that opts out of standard processing
public class LegacyUIStateComponent implements Component, DisableProcessingAssert {
    // ... component data
}
```

### Anti-Patterns (Do NOT do this)
- **New Implementations:** Do not implement this interface in any new component. This is a strong signal of incorrect design. The component should instead be integrated with the appropriate processing system or managed by a dedicated service.
- **Conditional Logic:** Do not write new systems that check for this interface. All new systems should rely on explicit component registration or other modern patterns. Refactor existing checks where possible.

## Data Pipeline
This component does not participate in any data pipeline. It is a passive marker used for control flow, not data flow. It neither receives, transforms, nor outputs data.

> Flow:
> N/A

