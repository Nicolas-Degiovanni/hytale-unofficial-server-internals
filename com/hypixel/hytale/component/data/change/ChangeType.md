---
description: Architectural reference for ChangeType
---

# ChangeType

**Package:** com.hypixel.hytale.component.data.change
**Type:** Enumeration / Type-Safe Constant

## Definition
```java
// Signature
public enum ChangeType {
```

## Architecture & Concepts
ChangeType is a fundamental enumeration that serves as a type-safe signal for state transitions within the data component system. Its primary role is to explicitly define the nature of a change event when a component is attached to or detached from an entity.

This enumeration is not a service or a manager; it is a static data contract. By using an enum instead of raw integers or strings, the system guarantees compile-time safety and eliminates a class of common bugs related to "magic values". It is a core building block for event-driven systems that react to the dynamic composition of game entities. Systems that listen for component changes, such as rendering, physics, or AI, rely on this type to correctly interpret the event payload and update their internal state.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine (JVM) when the ChangeType class is first loaded. This process is handled automatically by the ClassLoader.
- **Scope:** As static final instances, the constants REGISTERED and UNREGISTERED exist for the entire lifetime of the application. They are globally accessible and shared.
- **Destruction:** The constants are garbage collected only when the defining ClassLoader is unloaded, which typically happens at application shutdown.

## Internal State & Concurrency
- **State:** ChangeType instances are deeply immutable. Their state is defined at compile time and cannot be altered at runtime.
- **Thread Safety:** This enumeration is inherently thread-safe. As immutable, globally unique constants, they can be safely accessed and passed between any threads without synchronization or locking mechanisms. This is a critical guarantee for concurrent systems that may modify entity components from different threads.

## API Surface
The API surface consists solely of the predefined constants.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| REGISTERED | ChangeType | O(1) | Represents the event of a component being added to an entity. |
| UNREGISTERED | ChangeType | O(1) | Represents the event of a component being removed from an entity. |

## Integration Patterns

### Standard Usage
ChangeType is typically used as a field within a larger event object or as a parameter in a listener/callback interface to specify the action that occurred.

```java
// A listener system reacting to component changes
public class PhysicsSystem implements ComponentChangeListener {

    @Override
    public void onComponentChanged(Entity entity, Component component, ChangeType type) {
        if (component instanceof RigidBody) {
            if (type == ChangeType.REGISTERED) {
                // Add the entity to the physics simulation
                physicsWorld.addBody(entity);
            } else if (type == ChangeType.UNREGISTERED) {
                // Remove the entity from the physics simulation
                physicsWorld.removeBody(entity);
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Persistence via Ordinal:** Do not serialize or store the enum's ordinal value (e.g., `ChangeType.REGISTERED.ordinal()`). If the order of constants in the enum definition changes in a future update, all persisted data will become corrupt. Use `name()` for a more robust, albeit less performant, serialization strategy.
- **Switch without Default:** When using a switch statement on ChangeType, always include a default case to handle potential future additions to the enum, even if it only logs a warning. This prevents silent failures if new change types are introduced.

## Data Pipeline
ChangeType acts as a data flag that flows through the engine's event bus, originating from the core component management system.

> Flow:
> ComponentManager -> ComponentChangeEvent(..., type: **ChangeType**) -> EventBus -> System Listeners (e.g., PhysicsSystem, RenderSystem)

