---
description: Architectural reference for IAnnotatedComponent
---

# IAnnotatedComponent

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Contract Interface

## Definition
```java
// Signature
public interface IAnnotatedComponent {
```

## Architecture & Concepts
The IAnnotatedComponent interface is a foundational contract within the server-side NPC artificial intelligence framework. It establishes a standard for components that can be arranged in a hierarchical, tree-like structure, most commonly a behavior tree or a role-based configuration.

Its primary purpose is to enable introspection and debugging. Any class implementing this interface can report its position within the hierarchy (via getParent, getIndex), provide descriptive information about its state (via getInfo), and generate a human-readable "breadcrumb" trail for logging and diagnostics.

This interface is the cornerstone of the Composite design pattern as applied to NPC behaviors. It allows complex AI logic to be built from smaller, reusable, and introspectable parts, where both individual nodes (leaves) and composite nodes (branches) share a common interface.

## Lifecycle & Ownership
As an interface, IAnnotatedComponent does not have a lifecycle of its own. Instead, it defines the lifecycle contract that implementing objects must adhere to. The lifecycle of an object implementing this interface is dictated entirely by its container.

- **Creation:** Implementations are typically instantiated by a factory or builder when an NPC's behavior tree or Role is constructed. This often occurs during server startup or when an NPC is spawned into the world.
- **Scope:** The lifetime of an IAnnotatedComponent implementation is tightly coupled to its parent container, usually an NPC's specific Role instance. It persists as long as the NPC's AI controller is active and using that configuration.
- **Destruction:** The object is eligible for garbage collection when its parent NPC is despawned or its AI configuration is reloaded, causing the original tree of components to be discarded. There is no explicit destruction method defined in the contract.

## Internal State & Concurrency
- **State:** The interface contract implies that implementing classes will hold state. Specifically, they must maintain a reference to a parent component and an index, which are set via the setContext method. The getInfo method is designed to extract runtime state from the component. Therefore, implementations are expected to be **mutable**.
- **Thread Safety:** This interface is **not thread-safe**. All interactions with implementations of IAnnotatedComponent are expected to occur on the main server thread (the game tick). The setContext method modifies the internal state, and concurrent calls would lead to an inconsistent component tree.

**WARNING:** Do not access or modify an IAnnotatedComponent from asynchronous tasks or worker threads without explicit synchronization managed by the calling system. The NPC AI framework assumes single-threaded access.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInfo(Role, ComponentInfo) | void | O(1) | Populates a ComponentInfo object with the current state and metadata of this component. |
| setContext(IAnnotatedComponent, int) | void | O(1) | Establishes the component's position in the hierarchy by setting its parent and index. **Critical:** This should only be called once during initialization. |
| getParent() | IAnnotatedComponent | O(1) | Returns the parent component in the tree, or null if this is a root node. |
| getIndex() | int | O(1) | Returns the component's index relative to its siblings under the same parent. |
| getLabel() | String | O(1) | *Default Method.* Generates a simple, human-readable label for the component, including its index if available. |
| getBreadCrumbs() | String | O(N) | *Default Method.* Recursively traverses up the tree to the root, building a path-like string (e.g., Root|Branch|Leaf). N is the depth of the component in the tree. |

## Integration Patterns

### Standard Usage
This interface is not meant to be used directly but rather implemented by concrete AI components like actions, conditions, or decorators. A higher-level system, such as an AI executor or debugger, would traverse a tree of these components to perform operations.

```java
// Example of a system that inspects a component tree
public void inspectAiComponent(IAnnotatedComponent component) {
    // Retrieve the full hierarchical path for logging
    String path = component.getBreadCrumbs();
    System.out.println("Inspecting component at: " + path);

    // Gather detailed state information
    ComponentInfo info = new ComponentInfo();
    Role owningRole = findOwningRole(component); // Assumes a helper method exists
    component.getInfo(owningRole, info);

    // Process the gathered information
    System.out.println("Details: " + info.toString());
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Default Methods:** Do not override default methods like getBreadCrumbs with logic that relies on mutable instance state. Their behavior should be predictable and derived solely from the core contract methods (getParent, getIndex).
- **Cyclic Dependencies:** When calling setContext, never create a cyclical parent-child relationship (e.g., A is a parent of B, and B is a parent of A). This will cause a StackOverflowError when getBreadCrumbs is called.
- **Ignoring Context:** Do not implement this interface without correctly storing and returning the values provided by setContext. The integrity of the entire introspection system depends on it.

## Data Pipeline
IAnnotatedComponent does not participate in a traditional data processing pipeline. Instead, it facilitates a **query and introspection flow**. Data is pulled *from* the component on demand rather than pushed *through* it.

> Flow:
> AI Debugger / Executor -> Traverses Component Tree -> Calls **getBreadCrumbs()** / **getInfo()** on a specific **IAnnotatedComponent** -> Aggregates State -> Returns to Caller for Display or Logic<ctrl63>

