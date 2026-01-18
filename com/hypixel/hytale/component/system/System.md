---
description: Architectural reference for System
---

# System

**Package:** com.hypixel.hytale.component.system
**Type:** Base Class / Template

## Definition
```java
// Signature
public abstract class System<ECS_TYPE> implements ISystem<ECS_TYPE> {
```

## Architecture & Concepts
The System class is the foundational template for defining logical units of behavior within Hytale's Entity-Component-System (ECS) architecture. It is not a "system" in the traditional ECS sense of processing entities every frame. Instead, its primary and sole responsibility is **declarative registration**.

Subclasses of System act as manifests, defining the specific Component and Resource data types that exist within a given ECS context. During engine initialization, a discovery mechanism finds all concrete System implementations, instantiates them, and queries them for their registrations. This data is used to construct the master registries for the entire ECS world, enabling the engine to understand how to create, serialize, and manage all game data.

The generic parameter, ECS_TYPE, allows this registration pattern to be reused for different ECS "worlds," such as a client-side world, a server-side world, or specialized worlds for UI or other subsystems. This provides a clean, modular way to define the data schema for different parts of the application.

## Lifecycle & Ownership
The lifecycle of a System object is ephemeral and confined strictly to the engine's bootstrap phase.

-   **Creation:** A concrete subclass of System is instantiated once by a central SystemLoader or a similar bootstrap manager during application startup. The engine is responsible for discovering and creating these instances.
-   **Scope:** The object exists only for the duration of the registration process. Its purpose is fulfilled the moment its constructor completes and its registration lists are read by the engine.
-   **Destruction:** The System object is eligible for garbage collection immediately after the bootstrap manager has retrieved its component and resource definitions via getComponentRegistrations and getResourceRegistrations. It holds no persistent state and is not retained by any long-lived part of the engine.

## Internal State & Concurrency
-   **State:** The internal state consists of two mutable lists: componentRegistrations and resourceRegistrations. This state is populated exclusively during the object's construction phase via calls to the registerComponent and registerResource methods. After the constructor returns, the state should be considered effectively immutable.

-   **Thread Safety:** This class is **not thread-safe** and must not be treated as such. It is designed for a single-threaded, sequential bootstrap environment. All registration methods modify internal collections without any synchronization.

    **WARNING:** Concurrent access to a System instance will result in race conditions and a corrupted, unpredictable ECS registry. The entire registration process is architected to complete before any multi-threaded game logic begins.

## API Surface
The public and protected methods are designed for use by subclasses during their construction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerComponent(class, supplier) | ComponentType | O(1) | Declares a new Component type for the ECS world. Used for components that are not serialized. |
| registerComponent(class, id, codec) | ComponentType | O(1) | Declares a new Component type with a network ID and a codec for serialization. |
| registerResource(class, supplier) | ResourceType | O(1) | Declares a new global Resource type for the ECS world. |
| getComponentRegistrations() | List | O(1) | Called by the engine to retrieve the list of all declared Components from this System. |
| getResourceRegistrations() | List | O(1) | Called by the engine to retrieve the list of all declared Resources from this System. |

## Integration Patterns

### Standard Usage
The only correct way to use this class is to extend it. All registrations must occur within the constructor of the concrete subclass. The engine will handle the instantiation and data retrieval.

```java
// Example of a concrete system defining game components
public final class GameplaySystem extends System<ServerECS> {

    // Public handles to the registered types for other systems to use
    public final ComponentType<ServerECS, Health> health;
    public final ComponentType<ServerECS, Mana> mana;
    public final ResourceType<ServerECS, WorldTime> worldTime;

    // Registration occurs during construction
    public GameplaySystem() {
        this.health = registerComponent(Health.class, "hytale:health", Health.CODEC);
        this.mana = registerComponent(Mana.class, "hytale:mana", Mana.CODEC);
        this.worldTime = registerResource(WorldTime.class, WorldTime::new);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Late Registration:** Never call registerComponent or registerResource outside of the constructor. The engine reads the registration lists only once after instantiation. Any subsequent calls will have no effect, and the component will not exist in the ECS world.
-   **Manual Instantiation:** Do not create instances of your System subclasses manually (e.g., new GameplaySystem()). The engine's bootstrap process is responsible for managing their lifecycle. Manual instantiation is redundant and the instance will be ignored by the engine.
-   **Storing State:** Do not store any state in a System subclass beyond the final ComponentType and ResourceType handles. These objects are discarded after startup, and any other state will be lost.

## Data Pipeline
The System class is not part of the runtime data processing pipeline. It is a configuration provider at the very beginning of the engine's lifecycle.

> Flow:
> 1. Engine Startup
> 2. SystemLoader discovers and instantiates `GameplaySystem`
> 3. `GameplaySystem` constructor calls `registerComponent`
> 4. **System** base class adds registration data to its internal lists
> 5. SystemLoader calls `getComponentRegistrations()` on the `GameplaySystem` instance
> 6. Registration data is added to the global `ComponentRegistry`
> 7. The `GameplaySystem` instance is discarded
> 8. The `ComponentRegistry` is now used by the rest of the engine to manage game state.

