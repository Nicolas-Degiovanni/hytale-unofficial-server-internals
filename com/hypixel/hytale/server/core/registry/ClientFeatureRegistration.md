---
description: Architectural reference for ClientFeatureRegistration
---

# ClientFeatureRegistration

**Package:** com.hypixel.hytale.server.core.registry
**Type:** Transient

## Definition
```java
// Signature
public class ClientFeatureRegistration extends Registration {
```

## Architecture & Concepts
The ClientFeatureRegistration class is a descriptor object, not a service. Its primary role is to encapsulate the definition and lifecycle of a single server-side feature that is communicated to the client. It acts as a data-transfer object that couples a feature identifier, represented by the ClientFeature enum, with the logic required to manage its state.

This class is a foundational component of the server's feature registry system. Instead of having a central manager class with hardcoded feature logic, the system is designed to be extensible. Developers define a feature's behavior by creating an instance of ClientFeatureRegistration and submitting it to a central registry, likely the ClientFeatureHandler. This pattern decouples feature definition from feature management, allowing for dynamic addition or modification of features without altering the core registry service.

It inherits from the base Registration class, which provides the common contract for enabled checks and unregistration hooks used throughout the server's registry systems.

### Lifecycle & Ownership
- **Creation:** Instances are created at the point a server feature is defined, typically during server bootstrap or when a specific game module is initialized. It is not managed by a dependency injection container.
- **Scope:** The object's lifetime is tied directly to the registration of the feature it represents. It is held as a reference within the responsible registry service (e.g., ClientFeatureHandler) for as long as the feature is active.
- **Destruction:** The object is eligible for garbage collection once the feature is unregistered and all external references to the registration handle are released. The `unregister` Runnable provided at construction is executed by the registry upon removal, performing the necessary cleanup.

## Internal State & Concurrency
- **State:** The ClientFeatureRegistration object is immutable. Its primary state, the ClientFeature, is a final field set during construction. The lifecycle logic, provided as a BooleanSupplier and a Runnable, is also fixed upon instantiation. While the object itself is immutable, the *effective state* of the feature (i.e., the value returned by the `isEnabled` supplier) can change dynamically.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. It contains no internal locks or mutable fields. **Warning:** The thread safety of the overall feature management system depends entirely on the implementation of the `BooleanSupplier` and `Runnable` provided during construction. If these delegates access or modify shared state, they must be properly synchronized by the implementer.

## API Surface
The public API is minimal, focusing on construction and data retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ClientFeatureRegistration(ClientFeature) | Constructor | O(1) | Creates a registration that is always enabled and defaults to unregistering via ClientFeatureHandler. |
| ClientFeatureRegistration(ClientFeature, BooleanSupplier, Runnable) | Constructor | O(1) | Creates a registration with custom logic for its enabled state and unregistration behavior. |
| getFeature() | ClientFeature | O(1) | Returns the unique feature identifier this registration represents. |

## Integration Patterns

### Standard Usage
The intended use is to define a feature and submit it to the appropriate handler or registry service for management.

```java
// 1. Define the feature and its lifecycle logic
ClientFeature myFeature = ClientFeature.SOME_NEW_FEATURE;
BooleanSupplier isFeatureEnabled = () -> server.getConfig().isSomeNewFeatureActive();
Runnable unregisterLogic = () -> Logger.info("Unregistering SOME_NEW_FEATURE");

// 2. Create the registration object
ClientFeatureRegistration registration = new ClientFeatureRegistration(
    myFeature,
    isFeatureEnabled,
    unregisterLogic
);

// 3. Submit it to the managing service (hypothetical example)
server.getClientFeatureHandler().register(registration);
```

### Anti-Patterns (Do NOT do this)
- **Orphaned Instances:** Do not create a ClientFeatureRegistration instance without submitting it to a registry. The object itself has no side effects; it is merely a data container. An un-submitted instance is inert and will be garbage collected.
- **Stateful Lambdas:** Avoid providing complex, stateful lambdas for the `isEnabled` supplier or `unregister` runnable without ensuring their thread safety. The registry may invoke these from different threads, leading to race conditions if they access shared, mutable state without synchronization.

## Data Pipeline
ClientFeatureRegistration does not process data in a pipeline. Instead, it serves as a configuration object that is an *input* to the feature management system.

> Flow:
> Server Module Initialization -> **new ClientFeatureRegistration(...)** -> ClientFeatureHandler.register() -> Stored in Handler's internal collection -> Used during client handshake to report enabled features.

