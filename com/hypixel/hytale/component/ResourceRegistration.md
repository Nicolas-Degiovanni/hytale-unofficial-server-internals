---
description: Architectural reference for ResourceRegistration
---

# ResourceRegistration

**Package:** com.hypixel.hytale.component
**Type:** Value Object

## Definition
```java
// Signature
public record ResourceRegistration<ECS_TYPE, T extends Resource<ECS_TYPE>>(
   @Nonnull Class<? super T> typeClass,
   @Nullable String id,
   @Nullable BuilderCodec<T> codec,
   @Nonnull Supplier<T> supplier,
   @Nonnull ResourceType<ECS_TYPE, T> resourceType
) {
```

## Architecture & Concepts
ResourceRegistration is an immutable data carrier record that acts as a blueprint for defining a new Resource type within the engine's component system. It is not a service or a manager; rather, it is a descriptor object that bundles all the essential metadata required by a central registry to understand how to create, identify, and serialize a specific Resource.

This class is fundamental to the engine's data-driven and extensible architecture. By encapsulating the definition of a resource—including its class type, network/storage identifier, serialization logic (codec), and default factory (supplier)—it provides a standardized contract for both internal engine systems and external mods to introduce new data components into the game world. Its primary role is to facilitate the configuration phase of the resource management lifecycle.

## Lifecycle & Ownership
-   **Creation:** Instances are created transiently during the engine's bootstrap or a mod's initialization sequence. They are constructed directly using the canonical record constructor.
-   **Scope:** The lifecycle of a ResourceRegistration instance is exceptionally short. It is designed to be a temporary, pass-by-value object. It exists only long enough to be passed as an argument to a registration method, such as `ResourceManager.register`.
-   **Destruction:** Once the registration method has consumed the information, the ResourceRegistration object is no longer referenced and becomes eligible for garbage collection. There are no manual cleanup or destruction protocols.

**WARNING:** Holding a long-term reference to a ResourceRegistration instance is an anti-pattern. The authoritative definition of the resource is maintained by the registry that consumes this object, not the object itself.

## Internal State & Concurrency
-   **State:** As a Java record, ResourceRegistration is **deeply immutable**. All its fields are final and are assigned only at the time of construction. This design guarantees that a resource's definition cannot be altered after it has been declared.
-   **Thread Safety:** The immutable nature of this record makes it inherently **thread-safe**. An instance can be safely shared and read across multiple threads without any need for synchronization or locking mechanisms. This is critical during parallelized asset loading and system initialization.

## API Surface
The public API is implicitly defined by the record's components. It consists solely of the canonical constructor and accessor methods for each field (e.g., `typeClass()`, `id()`, `codec()`). There are no complex operations or behaviors associated with this object. Its purpose is to carry data, not to perform actions.

## Integration Patterns

### Standard Usage
The sole intended use case is to construct an instance and immediately pass it to a registration service. This is typically done within a static initializer block or an initialization entry point.

```java
// Example of registering a hypothetical "ManaComponent" resource
ResourceRegistration<Entity, ManaComponent> registration = new ResourceRegistration<>(
    ManaComponent.class,
    "hytale:mana",
    ManaComponent.CODEC,
    ManaComponent::new,
    MANA_RESOURCE_TYPE
);

// The registration is then consumed by a central registry
resourceRegistry.register(registration);
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage:** Do not store instances of ResourceRegistration in fields or collections for later use. They represent a point-in-time configuration payload, not a live object.
-   **Conditional Logic:** Do not build complex application logic around the state of a ResourceRegistration object. Query the authoritative resource manager or registry for resource information at runtime.
-   **Null Contract Violation:** Providing a null value for fields marked with @Nonnull (typeClass, supplier, resourceType) will lead to immediate runtime exceptions and system instability.

## Data Pipeline
ResourceRegistration is a key element in the system's *configuration pipeline*, not a runtime data processing pipeline. It is used to populate the systems that will later handle live data.

> Flow:
> Engine/Mod Initialization Code -> **new ResourceRegistration(...)** -> ResourceRegistry.register() -> Internal Registry Data Structures (e.g., Maps) -> Runtime Systems (e.g., Serializers, Factories)<ctrl63>

