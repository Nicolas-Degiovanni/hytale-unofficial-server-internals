---
description: Architectural reference for ComponentRegistration
---

# ComponentRegistration

**Package:** com.hypixel.hytale.component
**Type:** Value Object

## Definition
```java
// Signature
public record ComponentRegistration<ECS_TYPE, T extends Component<ECS_TYPE>>(
   @Nonnull Class<? super T> typeClass,
   @Nullable String id,
   @Nullable BuilderCodec<T> codec,
   @Nonnull Supplier<T> supplier,
   @Nonnull ComponentType<ECS_TYPE, T> componentType
) {
}
```

## Architecture & Concepts

The ComponentRegistration record is a foundational element of the Hytale Entity-Component-System (ECS) framework. It is not a service or an active participant in the game loop; rather, it is an immutable **descriptor** or **blueprint** that defines how a specific Component type is managed by the engine.

This class encapsulates all the essential metadata required for the ECS to dynamically instantiate, serialize, and identify a component at runtime. By bundling this information into a single, declarative object, the core ECS engine remains decoupled from concrete component implementations. This design allows developers to introduce new component types into the game without modifying the engine's core systems. The engine simply consumes a collection of these registrations during its initialization phase.

- The **typeClass** provides the runtime Java class for reflection and type checking.
- The **id** provides a human-readable, stable string identifier used for serialization, network synchronization, and data files.
- The **codec** provides the specific logic for encoding and decoding the component's state, bridging the gap between in-memory representation and a serialized format.
- The **supplier** acts as a factory, providing a standardized way for the ECS to create new instances of the component.
- The **componentType** is an internal, optimized handle used by the ECS for high-performance lookups and type management, avoiding the overhead of string or class comparisons in performance-critical code paths.

### Lifecycle & Ownership
- **Creation:** Instances are created declaratively during the engine's bootstrap or module loading phase. A central registry, such as a ComponentRegistry, is responsible for collecting these definitions from various parts of the codebase. They are considered static configuration data.
- **Scope:** A ComponentRegistration is a singleton in effect, though not in implementation. Once created and registered at startup, it persists for the entire application lifecycle.
- **Destruction:** These objects are not explicitly destroyed. They are garbage collected during final JVM shutdown when the application terminates.

## Internal State & Concurrency
- **State:** As a Java record, ComponentRegistration is fundamentally **immutable**. All its fields are final and are assigned only at construction. This guarantees that component definitions cannot be altered at runtime, preventing a wide class of state corruption bugs.
- **Thread Safety:** The record is inherently **thread-safe**. Its immutability allows it to be freely shared and read across any number of threads without requiring locks or other synchronization primitives.

**WARNING:** While the record itself is thread-safe, the objects it references (specifically the codec and supplier) must also be designed for concurrent access if the ECS engine operates in a multi-threaded environment.

## API Surface

The public API consists solely of the record's component accessors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| typeClass() | Class<? super T> | O(1) | Returns the Java class of the registered component. |
| id() | String | O(1) | Returns the unique, serializable identifier. Returns null if not specified. |
| codec() | BuilderCodec<T> | O(1) | Returns the serialization/deserialization logic. Returns null if not serializable. |
| supplier() | Supplier<T> | O(1) | Returns a factory function used to construct new component instances. |
| componentType() | ComponentType<ECS_TYPE, T> | O(1) | Returns the internal, optimized type handle for fast ECS lookups. |

## Integration Patterns

### Standard Usage

Developers do not typically interact with this class directly in gameplay code. Its primary use is during the static initialization of game modules, where components are defined and registered with the engine.

```java
// Example of registering a "PositionComponent" during engine startup
ComponentRegistry registry = getEngine().getComponentRegistry();

// The ComponentType is typically a pre-defined static handle
final ComponentType<Entity, PositionComponent> POSITION_TYPE = ...;

// The registration object is constructed and passed to the registry
registry.register(
    new ComponentRegistration<>(
        PositionComponent.class,
        "hytale:position",
        PositionComponent.CODEC,
        PositionComponent::new,
        POSITION_TYPE
    )
);
```

### Anti-Patterns (Do NOT do this)
- **Runtime Registration:** Do not attempt to register new component types after the engine has completed its bootstrap phase. The ECS is optimized based on a fixed set of components known at startup. Late registration can lead to undefined behavior and severe performance degradation.
- **Inconsistent Metadata:** Ensure that the `typeClass`, `codec`, `supplier`, and `componentType` all refer to the same logical component. A mismatch (e.g., a `PositionComponent.class` with a `VelocityComponent::new` supplier) will cause `ClassCastException`s and data corruption. The compiler's generic type checks provide a strong first line of defense against this.

## Data Pipeline

ComponentRegistration is not part of a runtime data pipeline; it is the **configuration** that defines the pipeline for a given component. The ECS engine uses the information within this record to correctly process component data.

> **Engine Usage Flow:**
> 1. **Engine Startup:** A central `ComponentRegistry` is populated with `ComponentRegistration` instances from all game modules.
> 2. **Entity Deserialization:** A network packet or save file contains data for a component with the ID "hytale:position".
> 3. **Lookup:** The engine queries the `ComponentRegistry` for the `ComponentRegistration` associated with that ID.
> 4. **Instantiation & Decoding:** The engine uses the retrieved registration's **supplier()** to create a new `PositionComponent` instance and its **codec()** to decode the serialized data into the new instance.
> 5. **Attachment:** The fully hydrated component is attached to the target entity.

