---
description: Architectural reference for AssetIconProperties
---

# AssetIconProperties

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Transient Data Model

## Definition
```java
// Signature
public class AssetIconProperties implements NetworkSerializable<com.hypixel.hytale.protocol.AssetIconProperties> {
```

## Architecture & Concepts
AssetIconProperties is a data-holding class that encapsulates the visual transformation properties for an asset's icon, typically used for rendering in inventories and user interfaces. It is not a service or manager, but rather a configuration object that represents a small, specific part of a larger asset's definition.

The class serves two primary functions:

1.  **Asset Deserialization Target:** The static **CODEC** field is the most critical architectural feature. It integrates this class with Hytale's reflection-based asset loading system. During server initialization, asset definition files (e.g., JSON) are parsed, and the data under keys such as *Scale*, *Translation*, and *Rotation* is automatically mapped into a new instance of AssetIconProperties.

2.  **Network Serialization Bridge:** By implementing the NetworkSerializable interface, this class acts as a bridge between the server-side configuration and the client-side protocol. The toPacket method converts the high-precision, server-side data types (like Vector2d) into a network-optimized, floating-point format (Vector2f, Vector3f) suitable for efficient transmission to game clients.

This object is fundamentally a data container, designed to be populated from a file and then converted into a network message.

### Lifecycle & Ownership
-   **Creation:** Instances are almost exclusively created by the Hytale asset loading framework via the static **CODEC** during server startup or when asset packs are loaded. Manual instantiation via the public constructor is possible but generally discouraged as it bypasses the standard asset pipeline.
-   **Scope:** The lifetime of an AssetIconProperties instance is strictly bound to its parent asset (e.g., an ItemAsset). It persists in memory as long as the parent asset is cached by the AssetManager. It is not a global or session-scoped object.
-   **Destruction:** Instances are marked for garbage collection when their parent asset is unloaded from the AssetManager. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** The internal state is **mutable** and consists of a float and two nullable vector objects. However, in practice, these objects should be treated as immutable after being populated by the asset loader. The class does not cache any data and its state directly reflects the loaded asset configuration.
-   **Thread Safety:** This class is **not thread-safe**. It is a plain data object with no internal locking or synchronization mechanisms. It is designed to be created and populated on a single thread (typically the main server thread or a dedicated asset loading thread) and subsequently read by game logic.

**WARNING:** Concurrent modification of an AssetIconProperties instance will lead to race conditions and unpredictable behavior. Do not modify its state after it has been loaded and integrated into the game's asset registry.

## API Surface
The public API is minimal, focusing on data access and network conversion.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.AssetIconProperties | O(1) | Converts the object into its network protocol representation. This is the primary mechanism for sending icon data to the client. |
| getScale() | float | O(1) | Returns the icon's uniform scale factor. |
| getTranslation() | Vector2f | O(1) | Returns the 2D screen-space translation, or null if not defined. |
| getRotation() | Vector3f | O(1) | Returns the 3D Euler rotation, or null if not defined. |

## Integration Patterns

### Standard Usage
Developers typically do not interact with this class directly. Instead, it is accessed as a property of a larger, parent asset retrieved from a central manager.

```java
// Correctly access properties through a parent asset
ItemAsset parentItem = assetManager.getAsset(ItemAsset.class, "my_namespace:my_cool_sword");
AssetIconProperties iconProps = parentItem.getIconProperties();

if (iconProps != null) {
    // Use the properties for server-side logic or rendering calculations
    float scale = iconProps.getScale();
    System.out.println("Icon scale is: " + scale);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Avoid `new AssetIconProperties()` in game logic. This creates a disconnected data object that is not part of the canonical asset system, leading to inconsistencies. Use this only for unit testing or highly specialized dynamic asset generation.
-   **Post-Load Modification:** Do not get an instance from an asset and then attempt to change its values. The asset system treats loaded data as a source of truth. Modifying it at runtime can cause desynchronization between the server state and what clients perceive.

## Data Pipeline
The primary flow for this class is from a configuration file on disk to a network packet sent to the client.

> Flow:
> Asset File (JSON) -> Asset Loading Service (using **CODEC**) -> **AssetIconProperties Instance** -> toPacket() -> Network Packet -> Game Client

