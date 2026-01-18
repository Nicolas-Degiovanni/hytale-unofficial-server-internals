---
description: Architectural reference for Reference
---

# Reference

**Package:** com.hypixel.hytale.builtin.hytalegenerator.referencebundle
**Type:** Utility (Marker Type)

## Definition
```java
// Signature
public class Reference {
```

## Architecture & Concepts
The Reference class is a foundational marker type used by the Hytale Generator toolchain. It serves as a common, type-safe base for auto-generated classes that represent bundles of assets or data. This class itself contains no logic or state; its entire purpose is to exist within the type system.

Architecturally, it acts as a contract between the code generation system and the runtime asset loading system. The generator produces subclasses of Reference, each corresponding to a specific resource bundle. The runtime can then discover and manage these bundles by identifying all types that inherit from Reference, ensuring that only valid, generator-produced handles are used to request bundled resources. This pattern avoids the use of error-prone string-based identifiers for critical asset groups.

**WARNING:** This class is not intended for direct use or extension by developers. It is a core component of an automated pipeline.

## Lifecycle & Ownership
- **Creation:** Instances of Reference subclasses are created by the Hytale Asset Management system when a specific bundle is requested and loaded into memory. The class definitions themselves are created at build-time by the Hytale Generator tool.
- **Scope:** The lifetime of a Reference instance is tied directly to the lifetime of the asset bundle it represents. It is ephemeral and exists only as long as the bundle is actively loaded.
- **Destruction:** Instances are eligible for garbage collection as soon as the associated asset bundle is unloaded and all references to the handle are released. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** Immutable. The base Reference class is stateless. Generated subclasses are also expected to be stateless handles.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. It can be safely passed between threads without synchronization, as is common when requesting assets from a worker thread.

## API Surface
The Reference class exposes no public API. Its value is derived entirely from its position in the class hierarchy.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| (none) | - | - | This class has no methods or fields. It is a marker type. |

## Integration Patterns

### Standard Usage
Developers do not interact with the Reference class directly. Instead, they use the specific, auto-generated subclasses as type-safe keys for loading resource bundles via a higher-level manager.

```java
// Correctly load a generated bundle using its specific type
// Note: MyGeneratedAssetBundle.class would be an auto-generated subclass of Reference
AssetBundle bundle = AssetManager.loadBundle(MyGeneratedAssetBundle.class);
Texture titleScreen = bundle.getTexture("title_screen");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance with `new Reference()`. It serves no purpose and will be rejected by engine systems.
- **Manual Subclassing:** Do not manually extend this class, as in `class MyCustomBundle extends Reference`. The asset system relies on metadata and code produced by the Hytale Generator, and manually created subclasses will not be recognized.
- **Type Checking:** Avoid using `instanceof Reference`. Use the specific generated subclass for all type-related logic to ensure correctness.

## Data Pipeline
The Reference class is not a data processor but a static identifier within a larger data generation and loading pipeline.

> Flow:
> Hytale Generator Tool -> Creates `GeneratedBundle.java extends Reference` -> Asset System uses `GeneratedBundle.class` as a key -> Loads physical bundle file -> Game Code receives handle to loaded assets

