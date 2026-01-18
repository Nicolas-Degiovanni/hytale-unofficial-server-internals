---
description: Architectural reference for UIComponentList
---

# UIComponentList

**Package:** com.hypixel.hytale.server.core.modules.entityui
**Type:** Transient Data Component

## Definition
```java
// Signature
public class UIComponentList implements Component<EntityStore> {
```

## Architecture & Concepts
The UIComponentList is a server-side data component within Hytale's Entity Component System (ECS) framework. Its sole responsibility is to associate a specific list of user interface elements with a game entity. It functions as a data-transfer object, holding the state that defines which UI components an entity possesses.

This component acts as a critical bridge between human-readable asset definitions and a performance-optimized runtime representation. It stores UI component references as an array of strings, which are resolved into a dense integer array of IDs during its lifecycle. This two-stage representation allows for flexible configuration in asset files while enabling high-performance lookups by other game systems at runtime.

The static CODEC field is central to its design, indicating that UIComponentList is fully serializable. This allows it to be persisted within the EntityStore as part of world save data and rehydrated when a chunk or region is loaded. The `afterDecode` lifecycle hook within the codec is a key architectural feature, ensuring that the component's internal state is synchronized with the server's master asset list immediately upon deserialization.

## Lifecycle & Ownership
-   **Creation:** A UIComponentList instance is created under two primary conditions:
    1.  **Deserialization:** The most common path. The static CODEC instantiates the object when an entity is loaded from the EntityStore (world storage).
    2.  **Programmatic Attachment:** A game system can programmatically create and attach this component to an entity at runtime to dynamically assign a UI.

-   **Scope:** The lifecycle of a UIComponentList instance is strictly bound to the entity to which it is attached. It persists as long as the parent entity exists within the server's world simulation.

-   **Destruction:** The component is marked for garbage collection when its parent entity is unloaded or destroyed. No manual cleanup is required.

## Internal State & Concurrency
-   **State:** The component's state is mutable. It maintains two core fields:
    -   *components*: A string array containing the names of the UI assets. This is the source of truth, primarily used for serialization.
    -   *componentIds*: An integer array that acts as a cached, resolved mapping of the string names. This cache is populated and potentially resized by the `update` method.

-   **Thread Safety:** **This class is not thread-safe.** All operations on ECS components, including this one, must be performed on the main server thread. The `update` method reads from a global, static asset map (`EntityUIComponent.getAssetMap`), which is a shared resource. Concurrent access would lead to severe data corruption and server instability.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the registered type identifier for this component from the EntityUIModule. |
| update() | void | O(N) | **Internal Use Only.** Synchronizes the internal `componentIds` cache with the global asset map. N is the total number of registered EntityUIComponents. |
| getComponentIds() | int[] | O(1) | Returns the cached array of resolved integer IDs for the attached UI components. |
| clone() | Component | O(1) | Creates a shallow copy of the component. Used by the ECS framework for entity duplication. |

## Integration Patterns

### Standard Usage
The component is typically accessed read-only by other systems to determine which UI elements to manage for a given entity. The framework handles its creation and update lifecycle automatically.

```java
// A hypothetical system that processes UI for an entity
void processEntityUI(Entity entity) {
    UIComponentList uiList = entity.getComponent(UIComponentList.class);

    if (uiList != null) {
        int[] ids = uiList.getComponentIds();
        // Use the integer IDs for fast lookups or processing
        for (int uiId : ids) {
            // ... perform logic based on the UI component ID
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new UIComponentList()`. Components must be added to entities via the appropriate entity management APIs, such as `entity.addComponent()`, to ensure they are correctly registered with the ECS.

-   **Manual Update Calls:** Avoid calling the `update()` method directly. The `afterDecode` hook in the serialization codec is designed to manage this process. Calling it unnecessarily, such as every game tick, will introduce significant performance overhead.

-   **Cross-Thread Access:** Reading or writing to a UIComponentList instance from any thread other than the main server thread is strictly forbidden and will lead to unpredictable behavior and crashes.

## Data Pipeline
The primary data flow for this component involves the transformation of asset names into runtime-efficient identifiers upon being loaded into the game world.

> Flow:
> World Storage (EntityStore) -> CODEC Deserialization (reads string array) -> **UIComponentList** Instance Created -> `afterDecode` Hook Triggers `update()` -> `componentIds` Cache Populated -> Game Systems Read `componentIds` for Processing

