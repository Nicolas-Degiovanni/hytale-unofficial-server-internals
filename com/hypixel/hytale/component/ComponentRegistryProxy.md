---
description: Architectural reference for ComponentRegistryProxy
---

# ComponentRegistryProxy

**Package:** com.hypixel.hytale.component
**Type:** Transient

## Definition
```java
// Signature
public class ComponentRegistryProxy<ECS_TYPE> implements IComponentRegistry<ECS_TYPE> {
```

## Architecture & Concepts
The ComponentRegistryProxy is an implementation of the Proxy design pattern, providing a managed, scoped, and reversible interface to a master ComponentRegistry. Its primary role within the engine is to facilitate safe registration and unregistration of ECS (Entity-Component-System) definitions for ephemeral or modular contexts, such as game mods, plugins, or temporary game states.

Instead of registering components directly with the global registry, a module interacts with a ComponentRegistryProxy. For every registration call (e.g., registerComponent, registerSystem), the proxy performs two actions:
1.  It forwards the registration request to the real, underlying ComponentRegistry.
2.  It records a corresponding "undo" or "cleanup" operation in an external list of consumers provided during its construction.

This architecture ensures that all registrations made through the proxy can be cleanly and completely reversed when the module is unloaded, preventing memory leaks, state corruption, and definition collisions. The proxy acts as a transactional layer for ECS definitions, guaranteeing that a module's lifecycle is isolated from the core engine state.

### Lifecycle & Ownership
-   **Creation:** A ComponentRegistryProxy is not a singleton and must not be created directly by feature developers. It is instantiated by a higher-level system, typically a module or asset loader. The creator supplies the global ComponentRegistry and a new, empty list which the proxy will populate with cleanup actions.
-   **Scope:** The proxy's lifetime is intentionally brief, tied directly to the initialization phase of the module it serves. It is a short-lived object used to capture a set of registration operations.
-   **Destruction:** The proxy itself is eligible for garbage collection as soon as the module's registration phase is complete. The critical artifact it produces—the list of unregister actions—is retained by the module loader and executed when the module is unloaded or reloaded. The empty shutdown method is by design; cleanup is managed externally.

## Internal State & Concurrency
-   **State:** The ComponentRegistryProxy is stateful, holding final references to the master registry and the list of unregister operations. Its primary state-mutating activity is adding new cleanup lambdas to this external list.
-   **Thread Safety:** **This class is not thread-safe.** ECS definition registration is a sensitive operation that must occur in a controlled, single-threaded context during specific engine lifecycle phases (e.g., startup, module loading). Attempting to register components through a shared proxy from multiple threads will result in race conditions and an unpredictable final state in both the master registry and the unregister list. All interactions must be externally synchronized or confined to a dedicated initialization thread.

## API Surface
The API mirrors the IComponentRegistry interface. All registration methods function as pass-throughs that additionally record a cleanup action.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerComponent(...) | ComponentType | O(1) | Delegates component registration to the master registry and records an unregister action. |
| registerResource(...) | ResourceType | O(1) | Delegates resource registration to the master registry and records an unregister action. |
| registerSystem(...) | void | O(1) | Delegates system instance registration and records an unregister action for its class. |
| registerSystemType(...) | SystemType | O(1) | Delegates system type registration and records an unregister action. |
| registerEntityEventType(...) | EntityEventType | O(1) | Delegates entity event type registration and records an unregister action. |
| registerWorldEventType(...) | WorldEventType | O(1) | Delegates world event type registration and records an unregister action. |
| shutdown() | void | O(1) | No-op. This method is intentionally empty. Cleanup is handled by the external system that created the proxy. |

## Integration Patterns

### Standard Usage
The proxy is intended to be used by a manager class that controls a module's lifecycle. The manager creates the proxy, passes it to the module for registration, and later uses the populated cleanup list to tear down the module.

```java
// In a hypothetical ModuleLoader service

// 1. Obtain the master registry from the engine core
ComponentRegistry globalRegistry = engine.getEcsManager().getRegistry();
List<BooleanConsumer> cleanupActions = new ArrayList<>();

// 2. Create a scoped proxy for the new module
IComponentRegistry moduleApi = new ComponentRegistryProxy<>(cleanupActions, globalRegistry);

// 3. The module uses the provided API to register its ECS definitions
MyModule module = new MyModule();
module.onLoad(moduleApi); 

// ... time passes, game runs ...

// 4. When the module is unloaded, execute the captured cleanup actions
// The 'false' argument signifies a hot-unload, not a full game shutdown.
for (BooleanConsumer action : cleanupActions) {
    action.accept(false);
}
cleanupActions.clear();
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Application or feature code should never construct a ComponentRegistryProxy with `new`. It should always be received from a context or module manager. Incorrectly managing the lifecycle of the `unregister` list will lead to severe memory leaks.
-   **Long-Term Storage:** Do not cache or store a reference to the ComponentRegistryProxy beyond the scope of a module's initialization phase. Its purpose is ephemeral.
-   **Manual Unregistration:** Do not manually unregister components that were registered through the proxy. The proxy's entire purpose is to automate this process. Doing so will cause errors when the automated cleanup logic is executed later.

