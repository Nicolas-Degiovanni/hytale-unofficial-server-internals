---
description: Architectural reference for AnnotatedComponentBase
---

# AnnotatedComponentBase

**Package:** com.hypixel.hytale.server.npc.corecomponents
**Type:** Component Base

## Definition
```java
// Signature
public abstract class AnnotatedComponentBase implements IAnnotatedComponent {
```

## Architecture & Concepts
AnnotatedComponentBase is the foundational abstract class for all server-side NPC components that participate in the annotation-driven composition system. It is not intended for direct instantiation but serves as the common ancestor for concrete behaviors, sensors, or other logic modules attached to an NPC.

The core architectural concept is the establishment of a **hierarchical component tree**. Each component instance is aware of its position within this structure through its *parent* and *index* properties. This allows for complex, ordered behaviors and enables components to traverse the hierarchy to communicate with siblings or ancestors.

This class provides the boilerplate implementation for context-awareness (parent and index), freeing concrete subclasses to focus solely on implementing their specific logic. It enforces the IAnnotatedComponent contract, signaling to the NPC entity system that subclasses can be discovered and managed automatically.

## Lifecycle & Ownership
-   **Creation:** Instances of subclasses are created by the NPC component management system, typically via reflection, when an NPC entity is spawned or its configuration is loaded. The system scans an NPC's definition for annotated fields and instantiates the corresponding component classes.
-   **Scope:** The lifecycle of a component is strictly tied to its parent NPC entity. It persists as long as the NPC is active in the world.
-   **Destruction:** The component is eligible for garbage collection when the parent NPC is despawned or the component is explicitly removed from the NPC's component hierarchy. There is no explicit destruction method; cleanup relies on the Java garbage collector.

## Internal State & Concurrency
-   **State:** The internal state, specifically the parent and index, is **mutable**. These fields are null upon construction and are populated post-instantiation via the setContext method. This two-phase initialization is a critical aspect of the component lifecycle.
-   **Thread Safety:** This class is **NOT thread-safe**. It contains no internal locking mechanisms. All interactions with a component and its state are expected to occur on the primary server thread responsible for the owning NPC's updates. Unsynchronized access from other threads will lead to race conditions and undefined behavior.

## API Surface
The public contract is designed for lifecycle management and hierarchical introspection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInfo(Role role, ComponentInfo holder) | void | O(1) | Populates a holder with metadata. Subclasses must override this to provide meaningful information. |
| setContext(IAnnotatedComponent parent, int index) | void | O(1) | **Lifecycle Critical.** Injects the component's context within the hierarchy. Called by the framework immediately after instantiation. |
| getParent() | IAnnotatedComponent | O(1) | Retrieves the parent component in the hierarchy. Returns null if this is a root component or if context has not been set. |
| getIndex() | int | O(1) | Retrieves the component's positional index relative to its parent. |

## Integration Patterns

### Standard Usage
Developers do not interact with AnnotatedComponentBase directly. The standard pattern is to extend it to create a new, specific NPC behavior or system. The framework handles the lifecycle.

```java
// A concrete component extending the base class
public class MyCustomBehavior extends AnnotatedComponentBase {
    @Override
    public void getInfo(Role role, ComponentInfo holder) {
        // Subclasses provide their specific metadata
        holder.add("Executes custom logic A");
    }

    public void performAction() {
        // Logic can safely access parent after initialization
        IAnnotatedComponent parent = getParent();
        if (parent != null) {
            // Interact with the parent or other parts of the hierarchy
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Accessing Context in Constructor:** Do not attempt to access the parent or index from a subclass constructor. These values are only guaranteed to be populated *after* the constructor completes and setContext has been invoked by the framework. This will result in a NullPointerException.

    ```java
    // BAD: This will throw a NullPointerException
    public class BadComponent extends AnnotatedComponentBase {
        public BadComponent() {
            // ERROR: getParent() returns null here!
            System.out.println("My parent is: " + getParent().toString());
        }
    }
    ```

-   **Manual Context Management:** Never call setContext manually. This method is reserved for the NPC component framework. Manually altering a component's context during runtime can corrupt the NPC's behavior tree and lead to unpredictable crashes.

## Data Pipeline

This class is a structural element, not a data processing one. Its role is within the **NPC Initialization Pipeline**.

> Flow:
> NPC Definition Asset -> Annotation Processor -> **new ConcreteComponent()** -> Framework calls **component.setContext(parent, index)** -> Component is added to NPC -> Component is now active and ready for game ticks.

