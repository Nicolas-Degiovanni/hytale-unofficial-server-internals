---
description: Architectural reference for CollisionModuleConfig
---

# CollisionModuleConfig

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient

## Definition
```java
// Signature
public class CollisionModuleConfig {
```

## Architecture & Concepts
The CollisionModuleConfig class is a passive Data Transfer Object (DTO) that defines the operational parameters for the server-side physics and collision detection system. It does not contain any logic itself; instead, it serves as a structured container for values loaded from external configuration sources.

Its primary architectural feature is the static **CODEC** field, an instance of BuilderCodec. This codec establishes a formal contract for serialization and deserialization, allowing the Hytale engine to automatically populate an instance of this class from a data source, such as a world configuration file in JSON or a similar format. This pattern decouples the collision system's behavior from hard-coded constants, enabling fine-tuning of physics parameters on a per-world or per-server basis without recompiling the engine.

This object is read by the CollisionModule during its initialization to configure internal algorithms, such as the maximum size of collision extents and debugging options.

## Lifecycle & Ownership
- **Creation:** An instance is created by the Hytale serialization framework when a server or world configuration is loaded. The framework uses the public static **CODEC** field to map configuration keys (e.g., "ExtentMax") to the corresponding fields in a new CollisionModuleConfig object. Direct instantiation is rare and typically reserved for unit testing.
- **Scope:** The object's lifetime is bound to the component that owns it, typically an instance of the CollisionModule. It is not a global singleton; a new instance may be created whenever the collision system is re-initialized or reconfigured.
- **Destruction:** The object is eligible for garbage collection as soon as its owning CollisionModule is destroyed or is provided with a new configuration object, releasing the reference to the old one.

## Internal State & Concurrency
- **State:** The state is entirely **Mutable**. All configuration properties can be modified after instantiation via public setters. The object acts as a simple property bag with no internal logic to protect its state.
- **Thread Safety:** This class is **not thread-safe**. It contains no synchronization mechanisms. It is designed to be populated on a single thread during a configuration loading phase.

    **WARNING:** It is critically unsafe to modify a CollisionModuleConfig instance after it has been passed to an active CollisionModule. Doing so can introduce severe race conditions, as physics worker threads may read the configuration values concurrently. The object should be treated as immutable after its initial setup.

## API Surface
The public API consists almost exclusively of standard getters and setters for its properties. The following method provides minor logical utility.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hasMinimumThickness() | boolean | O(1) | Returns true if a specific minimum thickness has been defined in the configuration, distinguishing it from the default null state. |

## Integration Patterns

### Standard Usage
Developers do not typically interact with this class directly in code. Instead, they define its values declaratively in a configuration file, which the engine then uses to create and inject the object into the collision system.

A hypothetical world configuration file might look like this:
```json
{
  "modules": {
    "collision": {
      "ExtentMax": 8.0,
      "DumpInvalidBlocks": true,
      "MinimumThickness": 0.05
    }
  }
}
```
The engine's module loader would use the **CODEC** to parse the `collision` object and construct a fully-formed CollisionModuleConfig instance.

### Anti-Patterns (Do NOT do this)
- **Post-Initialization Mutation:** Modifying the configuration object after the CollisionModule has started using it. This will lead to unpredictable physics behavior and potential crashes.
    ```java
    // DANGEROUS: Do not do this
    CollisionModule module = server.getCollisionModule();
    CollisionModuleConfig config = module.getConfig();
    config.setExtentMax(1000.0); // Causes race conditions
    ```
- **Manual Instantiation:** Creating an instance with `new CollisionModuleConfig()` in production code. This bypasses the centralized configuration system, making server behavior difficult to manage and debug. All configuration should flow from persistent data files.

## Data Pipeline
This class functions at the beginning of the data pipeline for the collision system, acting as the initial source of configuration.

> Flow:
> Configuration File (JSON) -> Hytale Codec Engine -> **CollisionModuleConfig** (Instance) -> CollisionModule (Initialization) -> Physics Simulation State<ctrl63>

