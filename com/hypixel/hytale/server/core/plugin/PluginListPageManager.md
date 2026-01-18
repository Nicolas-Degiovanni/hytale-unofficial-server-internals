---
description: Architectural reference for PluginListPageManager
---

# PluginListPageManager

**Package:** com.hypixel.hytale.server.core.plugin
**Type:** Singleton

## Definition
```java
// Signature
public class PluginListPageManager {
```

## Architecture & Concepts
The PluginListPageManager is a centralized dispatcher that implements a variation of the Observer (Pub/Sub) design pattern. Its primary role is to decouple the core plugin management subsystem from any UI or presentation layer components that need to react to changes in plugin state.

In this pattern, the PluginListPageManager acts as the **Subject**. UI components, represented by the PluginListPage class, act as **Observers**. When a UI element that displays a list of plugins is created, it registers itself with this manager. When a plugin is enabled, disabled, loaded, or unloaded, an external system (typically the PluginManager) notifies the PluginListPageManager of the event. The manager then broadcasts this change to all registered PluginListPage observers, ensuring that all views are synchronized with the authoritative plugin state.

This architecture prevents the core PluginManager from needing direct knowledge of UI components, promoting a clean separation of concerns between server logic and presentation.

The nested static class, SessionSettings, is a data-only Component used within Hytale's Entity-Component-System (ECS) framework. While defined within this file for namespacing and co-location, its lifecycle and management are handled by the ECS and EntityStore, not directly by the PluginListPageManager. It likely serves as a configuration flag attached to a session or world entity.

## Lifecycle & Ownership
-   **Creation:** The singleton instance is created via its public constructor. This is expected to happen exactly once during the server's initial bootstrap sequence. The constructor assigns itself to the static *instance* field, making it globally accessible.
-   **Scope:** As a singleton with a static instance, this object persists for the entire lifetime of the server process. Its internal list of *activePages* will grow and shrink as UI components are created and destroyed.
-   **Destruction:** The manager is not explicitly destroyed. It is garbage collected along with all other static objects when the Java Virtual Machine terminates at the end of the server's lifecycle.

## Internal State & Concurrency
-   **State:** The manager's primary state is the *activePages* list, which is a mutable collection of registered PluginListPage observers.
-   **Thread Safety:** This class is thread-safe. The use of a **CopyOnWriteArrayList** for the *activePages* list is a critical design choice. This collection type ensures that iterations (during notification broadcasts) are safe from concurrent modifications. Any registration or deregistration operation creates a new copy of the underlying array.

    **WARNING:** While this approach guarantees safety, it has performance implications. Writes (register/deregister) are expensive O(N) operations. This design assumes that UI pages are registered and deregistered infrequently, while notifications may occur more often and from various threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static PluginListPageManager | O(1) | Retrieves the global singleton instance. |
| registerPluginListPage(page) | void | O(N) | Adds a page to the notification list. Expensive due to CopyOnWrite. |
| deregisterPluginListPage(page) | void | O(N) | Removes a page from the notification list. Expensive due to CopyOnWrite. |
| notifyPluginChange(plugins, id) | void | O(N) | Broadcasts a plugin state change to all registered pages. |

## Integration Patterns

### Standard Usage
A UI component that displays plugin status must register itself upon creation and deregister itself upon destruction to prevent memory leaks.

```java
// In a UI component's initialization logic
PluginListPage myPage = new PluginListPage(...);
PluginListPageManager.get().registerPluginListPage(myPage);

// In the same component's cleanup/destruction logic
PluginListPageManager.get().deregisterPluginListPage(myPage);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new PluginListPageManager()` outside of the server's core initialization. Doing so will overwrite the static singleton instance, effectively disconnecting all previously registered observers and causing them to miss subsequent updates. Always use the static `get()` method.
-   **Leaked Observers:** Failure to call `deregisterPluginListPage` when a UI component is no longer in use will result in a memory leak. The PluginListPageManager will maintain a strong reference to the obsolete page, preventing it from being garbage collected and continuing to send it unnecessary notifications.

## Data Pipeline
The manager facilitates a notification flow rather than a data processing pipeline. The sequence of events is unidirectional and fans out to multiple consumers.

> Flow:
> Plugin State Change (e.g., user command) -> PluginManager -> **PluginListPageManager.notifyPluginChange()** -> (Fan-out) -> All registered PluginListPage instances receive the update.

