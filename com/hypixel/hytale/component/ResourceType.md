---
description: Architectural reference for ResourceType
---

# ResourceType

**Package:** com.hypixel.hytale.component
**Type:** Handle / Descriptor

## Definition
```java
// Signature
public class ResourceType<ECS_TYPE, T extends Resource<ECS_TYPE>> implements Comparable<ResourceType<ECS_TYPE, ?>> {
```

## Architecture & Concepts
The ResourceType class is a foundational component of the Hytale Entity-Component-System (ECS) architecture. It is not a component itself, but rather a lightweight, high-performance **handle** or **type identifier** for a specific component class (a class that extends Resource).

In a performance-critical ECS, looking up components by their Java Class object is unacceptably slow due to reflection and hash map overhead. The ResourceType serves as a direct replacement for this pattern. Each component class registered with a ComponentRegistry is assigned a unique, integer-indexed ResourceType. This integer index allows for extremely fast, O(1) array lookups when retrieving component data from an entity.

This object embodies the Flyweight pattern. A single ResourceType instance is created for each component class within a given registry, and this shared instance is used throughout the engine to refer to that component type. Its primary role is to bridge the gap between the developer-friendly Class-based API and the engine's internal, performance-oriented, index-based data structures.

## Lifecycle & Ownership
The lifecycle of a ResourceType is strictly controlled and is inseparable from its parent ComponentRegistry. Direct management by client code is a critical error.

-   **Creation:** ResourceType instances are created and initialized exclusively by a ComponentRegistry during the component registration phase. The package-private visibility of the `init` method enforces this. A new ResourceType is instantiated when a previously unknown Resource class is registered with the system.

-   **Scope:** A ResourceType is valid only for the lifetime of the specific ComponentRegistry instance that created it. It should be considered a session-scoped object, where a "session" is the valid lifetime of its parent registry (e.g., a server instance, a client session).

-   **Destruction:** The object is not destroyed in a traditional sense but is **invalidated**. The package-private `invalidate` method is called by the ComponentRegistry when its internal state is reset or destroyed. This action flags the handle as stale, preventing its use with a re-initialized or different registry, which would otherwise lead to memory corruption or critical state errors.

## Internal State & Concurrency
-   **State:** The object's state is effectively immutable from an external consumer's perspective. The core fields—registry, tClass, and index—are set once during initialization and never change. The single mutable field, `invalid`, is managed internally by the parent ComponentRegistry to control the object's lifecycle.

-   **Thread Safety:** This class is **not thread-safe** during its initialization or invalidation phases. These operations must be performed by the ComponentRegistry in a single-threaded, controlled environment, such as during engine startup or world loading. Once initialized and handed out to consumers, the object's fields can be safely read from any thread, as they are constant for its valid lifetime. However, consumers must be aware that a handle can be invalidated by another thread (e.g., the main game loop), making checks via the `validate` method necessary in multi-threaded contexts.

## API Surface
The public API is designed for validation and retrieval of the underlying type information.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRegistry() | ComponentRegistry | O(1) | Returns the parent registry that owns this type handle. |
| getTypeClass() | Class | O(1) | Returns the underlying Java Class for this component type. |
| getIndex() | int | O(1) | Returns the critical, fast-lookup integer index for this type. |
| validateRegistry(registry) | void | O(1) | Throws IllegalArgumentException if this handle belongs to a different registry. |
| validate() | void | O(1) | Throws IllegalStateException if this handle has been invalidated by its parent registry. |

## Integration Patterns

### Standard Usage
A ResourceType should always be retrieved from a ComponentRegistry. It is then used as a key to query entities or other ECS systems for component data.

```java
// Correctly obtain a handle from the authoritative registry
ComponentRegistry registry = context.getService(ComponentRegistry.class);
ResourceType<EcsContext, Position> positionType = registry.getType(Position.class);

// Use the handle to perform a fast component lookup on an entity
Position pos = entity.getComponent(positionType);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ResourceType()`. The constructor is package-private for a reason. The object will be uninitialized and invalid, leading to immediate exceptions. All handles must be sourced from a ComponentRegistry.

-   **Cross-Registry Contamination:** A ResourceType from one registry (e.g., a client-side registry) is meaningless to another (e.g., a server-side registry). The internal indices will not match. Attempting to use a handle across registry boundaries is a fatal error, which `validateRegistry` is designed to prevent.

-   **Caching Across Lifecycle Events:** Do not cache and reuse ResourceType handles across major engine events like a world unload/reload. The ComponentRegistry may be reset, invalidating all previous handles. Always re-query the registry for fresh handles after such events.

## Data Pipeline
ResourceType does not process a stream of data. Instead, it acts as a routing key or address used to request data from the core ECS data stores.

> Flow:
> Game Logic -> ComponentRegistry.getType(Component.class) -> **ResourceType** (Returned Handle) -> Entity.getComponent(ResourceType) -> Raw Component Data Array -> Component Instance

