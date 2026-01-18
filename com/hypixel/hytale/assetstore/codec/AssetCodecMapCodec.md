---
description: Architectural reference for AssetCodecMapCodec
---

# AssetCodecMapCodec

**Package:** com.hypixel.hytale.assetstore.codec
**Type:** Composite Codec / Factory

## Definition
```java
// Signature
public class AssetCodecMapCodec<K, T extends JsonAsset<K>> 
    extends StringCodecMapCodec<T, AssetBuilderCodec<K, T>> 
    implements AssetCodec<K, T> {
```

## Architecture & Concepts
The AssetCodecMapCodec is a foundational component of the asset deserialization pipeline. It acts as a high-level dispatcher or factory, responsible for converting raw data streams (JSON or BSON) into specific, concrete Java asset objects.

Its primary architectural role is to manage polymorphism in asset definitions. Game assets are rarely monolithic; an "Item" asset, for example, might have subtypes like "Weapon", "Armor", or "Consumable". This class solves the problem of determining which specific Java class to instantiate from a generic asset file.

It achieves this by inspecting a designated "discriminator" field within the source data (by default, a field named "Type"). The value of this field is used as a key to look up a specialized, pre-registered codec responsible for handling that particular asset subtype.

Furthermore, this class is deeply integrated with the engine's asset inheritance system. It orchestrates the process where an asset can be defined as a child of another, inheriting and overriding its properties. It manages the logic for finding and applying the parent asset's data before decoding the child's specific fields.

In summary, AssetCodecMapCodec is the central routing mechanism that connects the generic asset loading system to the specific logic required for each unique type of game asset.

## Lifecycle & Ownership
- **Creation:** An AssetCodecMapCodec is instantiated during the engine's bootstrap or module initialization phase. A single instance is typically created for each major, polymorphic asset category (e.g., one for all Blocks, one for all Items). During this phase, its `register` method is called repeatedly to populate its internal map with all known subtypes.
- **Scope:** The configured instance is a long-lived object. It is typically stored in a central registry or service locator and persists for the entire application session. Its state (the registered codecs) is established at startup and is not expected to change during runtime.
- **Destruction:** The object is eligible for garbage collection when the engine shuts down and the central registries are cleared. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state is mutable during the initial configuration phase. The primary state consists of the internal `idToCodec` map, which maps string identifiers to their corresponding child codecs. After the bootstrap phase, this state should be considered effectively immutable. The class itself does not cache decoded assets.

- **Thread Safety:** This class is **not thread-safe for mutation**. All calls to `register` must be performed from a single thread during the application's initialization sequence. Attempting to register new codecs from multiple threads or after initialization will lead to race conditions and unpredictable behavior.

  However, the decoding methods (e.g., `decodeAndInheritJsonAsset`) are thread-safe for execution, provided that the underlying child codecs they delegate to are also thread-safe. The internal map lookup is a read-only operation that can be safely performed by multiple worker threads in the asset loading system.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(id, class, codec) | AssetCodecMapCodec | O(1) | **Critical Configuration Method.** Binds a string identifier (e.g., "Weapon") to a specific codec responsible for that asset type. Must be called during initialization. |
| decodeAndInheritJsonAsset(reader, parent, extraInfo) | T | O(N) | The primary deserialization entry point. Reads the discriminator key, finds the correct child codec, and orchestrates the full instantiation, inheritance, and decoding process. N is the size of the input data. |
| decodeAndInherit(document, parent, extraInfo) | T | O(N) | BSON-based equivalent of the JSON decoding method. |
| getKeyCodec() | KeyedCodec | O(1) | Provides access to the codec responsible for handling the primary "Id" field of an asset. |
| getParentCodec() | KeyedCodec | O(1) | Provides access to the codec responsible for handling the "Parent" field used for inheritance. |

## Integration Patterns

### Standard Usage
The standard pattern is to instantiate, configure, and register the codec in a central location during startup. It is never used directly for decoding; rather, it is passed to a higher-level system like an AssetManager.

```java
// In an initialization routine (e.g., BlockRegistry.java)

// 1. Create the master codec for all "Block" assets.
//    The key type is String (for block IDs like "hytale:stone").
AssetCodecMapCodec<String, Block> blockCodec = new AssetCodecMapCodec<>(
    StringCodec.INSTANCE,
    Block::setId,
    Block::getId,
    Block::setAssetData,
    Block::getAssetData
);

// 2. Register the specific codecs for each block subtype.
//    The string "hytale:basic" maps to the BasicBlockCodec.
blockCodec.register("hytale:basic", BasicBlock.class, new BasicBlockCodec());
blockCodec.register("hytale:glowing", GlowingBlock.class, new GlowingBlockCodec());
blockCodec.register("hytale:slab", SlabBlock.class, new SlabBlockCodec());

// 3. Register the fully configured master codec with a central system.
//    The AssetManager will now use this codec to load any file with "Type": "hytale:basic", etc.
codecRegistry.register(Block.class, blockCodec);
```

### Anti-Patterns (Do NOT do this)
- **Late Registration:** Do not call `register` after the codec has been handed off to the AssetManager or any other system. This is a severe race condition. All registrations must occur in a single-threaded, deterministic bootstrap phase.
- **Per-Asset Instantiation:** Do not create a new AssetCodecMapCodec for each asset you want to load. A single, shared instance should be configured once and reused for all assets of that category.
- **Incorrect Discriminator:** Ensure the discriminator key used in the constructor (e.g., "Type") matches the key present in the JSON/BSON asset files. A mismatch will prevent the system from selecting the correct subtype codec.

## Data Pipeline
The AssetCodecMapCodec acts as the primary dispatcher in the asset loading data flow. It receives a raw data stream and, based on its content, routes the decoding task to the appropriate specialized handler.

> Flow:
> JSON File on Disk -> RawJsonReader Stream -> **AssetCodecMapCodec** (Reads "Type" field) -> Dispatches to specific *SlabBlockCodec* -> New SlabBlock Object -> AssetManager Cache

