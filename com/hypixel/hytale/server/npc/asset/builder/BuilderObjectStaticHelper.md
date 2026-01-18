---
description: Architectural reference for BuilderObjectStaticHelper
---

# BuilderObjectStaticHelper

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient

## Definition
```java
// Signature
public class BuilderObjectStaticHelper<T> extends BuilderObjectReferenceHelper<T> {
```

## Architecture & Concepts
The BuilderObjectStaticHelper is a specialized component within the server's asset construction framework. It serves as a strict policy enforcer for configuration objects that are declared as *static*. It inherits from the more general BuilderObjectReferenceHelper but overrides key behaviors to enforce a much stricter contract.

Architecturally, this class ensures that certain foundational asset definitions—such as base NPC archetypes or fundamental material properties—are **fully self-contained and immutable**. By disallowing file or internal references, it prevents complex, multi-file dependency chains for assets that must be simple and predictable. This guarantees that a "static" object's definition exists entirely within its own JSON block, improving data integrity and simplifying the debugging of asset configurations.

Its primary role is not to build objects, but to first *validate* that a given configuration adheres to the static contract before proceeding with the build.

### Lifecycle & Ownership
- **Creation:** Instantiated internally by the BuilderManager when it parses a JSON configuration that is identified as a static definition. It is not intended for direct creation by developers.
- **Scope:** The instance is ephemeral. It exists only for the duration of the validation and construction of a single static object from a specific JSON block.
- **Destruction:** The object is eligible for garbage collection immediately after the `readConfig` and `staticBuild` methods complete and the resulting asset is passed to the caller, typically the Asset Registry.

## Internal State & Concurrency
- **State:** This class holds mutable state inherited from its parent, which includes parsed data from the JSON configuration. This state is populated during the `readConfig` call and is essential for the final `staticBuild` operation.

- **Thread Safety:** **This class is not thread-safe.** It is designed to operate within a single-threaded asset loading pipeline. Its internal state is modified sequentially during the parsing process. Concurrent calls to `readConfig` or `staticBuild` on the same instance will result in unpredictable behavior and likely data corruption or erroneous validation exceptions.

    **WARNING:** The entire asset builder framework should be treated as single-threaded to ensure deterministic asset loading.

## API Surface
The public API is minimal and intended for use only by the controlling BuilderManager.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(...) | void | O(N) | Parses the provided JsonElement. Throws IllegalStateException if the configuration violates the static contract (e.g., is not final or contains references). |
| staticBuild(manager) | T | O(C) | Constructs and returns the final object of type T. Throws exceptions if the object's constructor fails. Complexity depends on the target object's constructor. |

## Integration Patterns

### Standard Usage
This class is used implicitly by the asset system. A content designer or developer interacts with it by defining an object in a JSON file and marking it appropriately. The BuilderManager then selects this helper to process the definition.

A typical interaction is declarative, not programmatic.

**Hypothetical JSON Asset Definition:**
```json
{
  "type": "hytale:npc_archetype",
  "isFinal": true, // This flag is critical
  "components": {
    "health": 100,
    "name": "Static Base Golem"
  }
}
```

**Internal Engine Usage (Simplified):**
```java
// The BuilderManager would internally select and use the helper
// This code is conceptual and does not represent a real implementation.

JsonElement golemConfig = parseFile("golem.json");
BuilderObjectStaticHelper<NpcArchetype> helper = new BuilderObjectStaticHelper<>(NpcArchetype.class, context);

// The helper validates that no references exist and isFinal is true
helper.readConfig(golemConfig, builderManager, params, validationHelper);

// The helper builds the final, self-contained object
NpcArchetype archetype = helper.staticBuild(builderManager);
assetRegistry.register(archetype);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class directly using `new`. The BuilderManager is solely responsible for its lifecycle. Direct creation bypasses the framework's context and will lead to runtime failures.
- **Misconfiguration:** Attempting to define a reference within a static object's configuration will trigger an immediate `IllegalStateException`. This is by design and indicates a configuration error that must be fixed in the JSON file.

    **Invalid JSON Example:**
    ```json
    {
      "isFinal": true,
      "components": {
        "health": 100,
        "model": {
          "$ref": "models/golem.json" // This will throw an exception
        }
      }
    }
    ```
- **Ignoring Exceptions:** The exceptions thrown by this class are critical validation failures. They must not be caught and ignored, as this would result in a partially-initialized or corrupt asset being loaded into the game.

## Data Pipeline
The BuilderObjectStaticHelper acts as a validation gate and terminal build step within the broader asset loading pipeline.

> Flow:
> Asset JSON File -> `BuilderManager` (reads file) -> (Identifies as static) -> **`BuilderObjectStaticHelper`** (Validates no references) -> (Builds object `T`) -> Asset Registry

---

