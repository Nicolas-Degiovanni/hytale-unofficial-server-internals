---
description: Architectural reference for AssetHolder
---

# AssetHolder

**Package:** com.hypixel.hytale.assetstore
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface AssetHolder<K> {
}
```

## Architecture & Concepts
The AssetHolder interface serves as a foundational contract within the Hytale Asset Management subsystem. It establishes a common type for all objects designed to hold, own, or manage a reference to a game asset. Its primary architectural purpose is to enable polymorphism; systems like the AssetManager or the rendering engine can operate on a generic AssetHolder without needing to know the specifics of the underlying asset or its container.

This interface is a generic type, parameterized by K, which represents the key or identifier used to look up the asset (e.g., a String path, a numeric ID). While the base interface defines no methods, it acts as a "marker" interface, signaling an object's role in the asset lifecycle. Concrete implementations are expected to extend this contract with specific methods for accessing, loading, or unloading their managed asset.

## Lifecycle & Ownership
As an interface, AssetHolder itself does not have a lifecycle. Instead, it defines a behavioral contract that implementing classes must adhere to. The lifecycle of an object implementing AssetHolder is dictated entirely by its concrete class and the system that creates it.

- **Creation:** An AssetHolder implementation is typically instantiated by a factory or a higher-level manager in response to an asset request. For example, a TextureHolder might be created by the AssetManager when a model requires a specific texture.
- **Scope:** The scope is variable. An AssetHolder might be transient, existing only for the duration of a single frame's render, or it could be long-lived, persisting as long as a game object or UI screen is active.
- **Destruction:** Ownership and destruction are critical. The system that creates an AssetHolder is generally responsible for its cleanup. Destruction of the holder should trigger the release of its reference to the underlying asset, allowing the AssetManager's reference counting or garbage collection mechanisms to reclaim memory if the asset is no longer in use.

## Internal State & Concurrency
The AssetHolder interface is stateless. All state management is the responsibility of the implementing class.

- **State:** Concrete implementations will be stateful. They will at a minimum hold a reference to an asset and its key K. They may also cache data, track loading status (e.g., PENDING, LOADED, FAILED), or manage reference counts.
- **Thread Safety:** The interface makes no guarantees about thread safety. Implementations that can be accessed from multiple threads (e.g., a loading thread and the main game thread) must implement their own synchronization mechanisms. It is generally unsafe to assume an AssetHolder implementation is thread-safe unless explicitly documented as such.

## API Surface
The base interface defines no public methods. Its API contract is fulfilled by the methods defined in its concrete implementations. Sub-interfaces or classes are expected to provide methods for asset retrieval, status checking, and resource management.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| *(N/A)* | *(N/A)* | *(N/A)* | This interface specifies no methods. The API is defined by subtypes. |

## Integration Patterns

### Standard Usage
The primary pattern is to implement this interface on classes that encapsulate a single asset or a collection of related assets. This allows the class to be registered with and managed by the central AssetManager.

```java
// A concrete implementation holding a Texture asset
public class TextureHolder implements AssetHolder<String> {
    private Texture texture;
    private String assetKey;

    // ... implementation ...

    public Texture getTexture() {
        // Logic to ensure texture is loaded before returning
        return this.texture;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Misuse as a Generic Container:** Do not use AssetHolder for objects that are not managed by the asset subsystem. Applying this interface to general data containers dilutes its purpose and can mislead the static analysis tools and other developers.
- **Ignoring the Generic Type:** Implementing AssetHolder without specifying a concrete type for the key K is a design error. The key is essential for the AssetManager to perform lookups and manage the asset lifecycle.

## Data Pipeline
The AssetHolder is not an active participant in the data pipeline but rather a passive container that represents a terminal state or a cache within the pipeline. It holds the result of an asset loading operation.

> Flow:
> Asset Request (Key K) -> AssetManager -> AssetLoader -> Raw Data (Bytes) -> AssetFactory -> Asset Object -> **AssetHolder Implementation** -> Game System (e.g., Renderer)

