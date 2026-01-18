---
description: Architectural reference for PortalDescription
---

# PortalDescription

**Package:** com.hypixel.hytale.server.core.asset.type.portalworld
**Type:** Data Model / Asset Type

## Definition
```java
// Signature
public class PortalDescription {
```

## Architecture & Concepts
The PortalDescription class is a passive data model that represents the descriptive metadata for a portal world. It is not a service or manager; instead, it serves as a strongly-typed, immutable container for configuration data loaded from server asset files.

Its structure and deserialization logic are defined by the static **CODEC** field, a BuilderCodec instance. This codec is the contract that maps keys from a data source (e.g., a JSON file) to the private fields of this class. This pattern centralizes the serialization logic and decouples the data representation from the underlying file format.

PortalDescription acts as the definitive source for UI-related information about a portal, such as its translatable name, flavor text, theme color, and associated imagery. Game systems query this object to populate user interfaces and display contextual information to the player before they enter a new world.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the engine's codec system during the asset loading process. An asset manager reads a portal world's configuration file, finds the relevant data section, and invokes the static CODEC to deserialize the data into a new PortalDescription object. Manual instantiation is not supported and will result in a non-functional object.
-   **Scope:** The lifetime of a PortalDescription instance is bound to the lifecycle of the asset it belongs to. It persists in memory as long as the corresponding portal world is loaded and referenced by the server's asset registry.
-   **Destruction:** The object is marked for garbage collection when the owning asset is unloaded or the server shuts down. It has no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** The state is **effectively immutable**. After the codec populates its fields during creation, the object's state does not change. There are no public setters or methods that mutate internal fields. Getters that return collections provide defensive copies or unmodifiable views to protect the internal arrays.
-   **Thread Safety:** This class is **thread-safe for read operations**. Due to its immutable nature post-initialization, multiple threads can safely call its getter methods concurrently without requiring external synchronization.

## API Surface
The public API consists entirely of accessors for retrieving portal metadata.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDisplayName() | Message | O(1) | Returns a translatable Message object for the portal's name. |
| getFlavorText() | Message | O(1) | Returns a translatable Message object for the portal's description. |
| getThemeColor() | Color | O(1) | Returns the primary theme color associated with the portal. |
| getPillTags() | List<PillTag> | O(1) | Returns a list of cosmetic UI tags. Returns an empty list if none are defined. |
| getObjectivesKeys() | String[] | O(1) | Returns an array of translation keys for the portal's objectives. |
| getWisdomKeys() | String[] | O(1) | Returns an array of translation keys for helpful tips about the portal. |
| getSplashImageFilename() | String | O(1) | Returns the filename for the portal's splash screen image asset. |

## Integration Patterns

### Standard Usage
A PortalDescription object should always be retrieved from a higher-level manager or from its parent asset. It is never created or managed directly.

```java
// Correctly retrieve the description from a parent object (e.g., a PortalWorld)
PortalWorld myPortal = worldRegistry.getPortal("adventure_zone_1");
PortalDescription description = myPortal.getDescription();

// Use the data to configure UI elements
uiPanel.setTitle(description.getDisplayName());
uiPanel.setBackgroundColor(description.getThemeColor());
uiPanel.setSplashImage(assetManager.load(description.getSplashImageFilename()));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new PortalDescription()`. The resulting object will be empty and all getters will return null or empty collections, leading to NullPointerExceptions in downstream systems. The only valid creator is the engine's codec.
-   **State Modification:** Do not use reflection to modify the private fields of a PortalDescription instance after it has been created. The engine assumes these objects are immutable, and violating this contract can cause unpredictable UI rendering and state corruption across the client and server.

## Data Pipeline
PortalDescription is a terminal point in the asset loading pipeline, transforming raw configuration data into a usable, in-memory object for consumption by game systems.

> Flow:
> Portal Asset File (JSON) -> Server AssetManager -> BuilderCodec Deserializer -> **PortalDescription Instance** -> Game Logic & UI Systems

