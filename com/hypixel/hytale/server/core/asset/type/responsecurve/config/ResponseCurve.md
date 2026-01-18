---
description: Architectural reference for ResponseCurve
---

# ResponseCurve

**Package:** com.hypixel.hytale.server.core.asset.type.responsecurve.config
**Type:** Managed Asset

## Definition
```java
// Signature
public abstract class ResponseCurve implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, ResponseCurve>> {
```

## Architecture & Concepts

The ResponseCurve class is an abstract base for data-driven mathematical functions loaded from JSON assets. It serves as a fundamental building block within the game engine for systems requiring configurable, non-linear value mapping, such as AI behavior, damage falloff, or animation easing.

Architecturally, ResponseCurve is a polymorphic asset type. A central codec, **AssetCodecMapCodec**, is responsible for deserializing JSON data into concrete implementations like **ExponentialResponseCurve** or **LogisticResponseCurve** based on a type identifier within the asset file. This allows game designers to define complex mathematical behaviors in simple data files without requiring code changes.

All ResponseCurve instances are managed by a dedicated, static **AssetStore**. This store acts as a centralized, singleton-like registry, providing a global access point for all loaded curves. This design ensures that memory is managed efficiently and that all game systems reference the same canonical instance of a given curve.

A critical feature is the use of **WeakReference** and the nested **Reference** class. This pattern is a deliberate memory optimization. The engine can hold a **ResponseCurve.Reference** instead of a direct, strong reference. If the system comes under memory pressure and no other strong references to the curve exist, the garbage collector can reclaim it. The **Reference** class can then transparently re-fetch the asset from the **AssetStore** on next access, preventing null pointer exceptions while minimizing the resident memory footprint.

## Lifecycle & Ownership

-   **Creation:** ResponseCurve instances are **never** created directly using their constructor. They are instantiated exclusively by the Hytale asset loading pipeline during server or client startup. The **AssetCodecMapCodec** reads a JSON asset, determines the concrete type, and constructs the object, populating it with the defined parameters.
-   **Scope:** An instance of a ResponseCurve persists for the entire application session, cached within its static **AssetStore**. Its lifetime is tied to the lifetime of the asset registry itself.
-   **Destruction:** Objects are eligible for garbage collection when the **AssetStore** is cleared, which typically occurs during a full asset reload or application shutdown. The weak referencing system means an individual curve might be collected sooner if it is not actively in use.

## Internal State & Concurrency

-   **State:** The state of a ResponseCurve is considered **immutable** after it has been deserialized from its asset file. All parameters defining the curve's shape are loaded once and are not intended to be modified at runtime. The class itself caches no runtime data beyond its definitional parameters.
-   **Thread Safety:** The core method, **computeY**, is a pure function of its input and immutable internal state, making it inherently thread-safe. However, the static **getAssetStore** method contains a non-synchronized lazy initialization block.

    **Warning:** While the engine's bootstrap process is typically single-threaded, calling **getAssetStore** from multiple threads concurrently before it has been initialized is not safe and could result in undefined behavior. All subsequent read operations on the initialized asset store are thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the global, singleton store for all ResponseCurve assets. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Provides direct access to the underlying map of all loaded curves. |
| computeY(double) | abstract double | O(1) | The core function. Computes the output value (Y) for a given input (X). |
| Reference.get() | ResponseCurve | O(1) | Resolves the reference, re-loading from the AssetStore if the weak reference was cleared. |

## Integration Patterns

### Standard Usage

The correct pattern for holding a long-term reference to a ResponseCurve is to use the nested **Reference** class. This respects the engine's memory management strategy.

```java
// Obtain the asset map once
IndexedLookupTableAssetMap<String, ResponseCurve> curveMap = ResponseCurve.getAssetMap();

// Create a reference to a specific curve by its index and instance
// In a real system, the index and curve would be retrieved from a config or another asset
int curveIndex = curveMap.getIndex("hytale:damage_falloff_bow");
ResponseCurve curve = curveMap.getAsset(curveIndex);
ResponseCurve.Reference curveRef = new ResponseCurve.Reference(curveIndex, curve);

// Later, in a performance-critical loop
double input = 0.75;
double falloff = curveRef.get().computeY(input);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ExponentialResponseCurve()`. This bypasses the asset management system, creating an unmanaged object that will not be available to other systems and may cause memory leaks.
-   **Holding Strong References:** Avoid storing a direct, strong reference to a ResponseCurve in long-lived objects. This defeats the purpose of the weak reference system and may prevent the asset from being unloaded under memory pressure. Use **ResponseCurve.Reference** instead.
-   **Modifying State:** Do not use reflection or other means to modify the internal state of a ResponseCurve after it has been loaded. They are designed to be immutable assets.

## Data Pipeline

The data for a ResponseCurve flows from a design-time configuration file to a runtime computational result.

> Flow:
> JSON Asset File -> Asset Loading System -> **AssetCodecMapCodec** -> **ResponseCurve Instance** (in AssetStore) -> Game System requests **Reference.get()** -> Game System calls **computeY(input)** -> Calculated double value

