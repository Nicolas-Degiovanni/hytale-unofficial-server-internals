---
description: Architectural reference for DiagramCraftingBench
---

# DiagramCraftingBench

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config.bench
**Type:** Data Model

## Definition
```java
// Signature
public class DiagramCraftingBench extends CraftingBench {
```

## Architecture & Concepts
The DiagramCraftingBench class is a specialized data model representing a specific type of crafting station configuration. It does not introduce new logic but instead serves as a concrete type for deserialization, enabling the engine to differentiate between various crafting bench behaviors defined in game assets.

Its primary architectural role is to integrate with the Hytale asset loading pipeline via its static CODEC field. This BuilderCodec instance instructs the engine on how to parse a corresponding asset file (e.g., a JSON or HOCON file) and construct a DiagramCraftingBench object. By extending CraftingBench, it inherits a foundational data structure, and the codec system uses the parent's codec to parse shared properties before handling any specialized fields.

This class is a key example of Hytale's data-driven design. The existence of this type allows game designers to define a "diagram-based" crafting bench in a data file, and the engine can polymorphically handle it as a CraftingBench while still being able to identify its specific subtype when necessary.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale Codec system during the server's asset loading phase. The static CODEC field is invoked when the Asset Manager encounters a block asset configured to use this specific bench type.
- **Scope:** An instance of DiagramCraftingBench is tied to the lifecycle of the asset it represents. It is loaded once at server startup and persists in a central asset registry for the entire server session. It is effectively a singleton per unique asset definition.
- **Destruction:** The object is garbage collected when the server shuts down or during a full asset reload, at which point the central asset registries are cleared.

## Internal State & Concurrency
- **State:** As a configuration object loaded from static game files, an instance of DiagramCraftingBench is intended to be **immutable** post-initialization. All its state is inherited from the parent CraftingBench class and is populated once during deserialization.
- **Thread Safety:** The object is **conditionally thread-safe**. It is safe for concurrent reads from any thread after it has been fully constructed and published by the Asset Manager. Writing to this object at runtime is a severe anti-pattern and will lead to unpredictable and inconsistent game state. All initialization is performed on a single, dedicated asset loading thread.

## API Surface
This class defines no new public methods. Its primary public contract is the static CODEC used for its construction. All functional APIs are inherited from its parent, CraftingBench.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | N/A | **Critical:** The static codec used by the engine's asset system to deserialize this object from configuration files. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation or use. Instead, game systems retrieve it from a higher-level manager or as part of a larger configuration object, such as a BlockType.

```java
// Example of a system retrieving the bench configuration from a block
BlockType stoneTable = world.getBlockRegistry().get("hytale:stone_crafting_table");
CraftingBench benchConfig = stoneTable.getComponent(CraftingBench.class);

if (benchConfig instanceof DiagramCraftingBench) {
    // Logic specific to diagram-based crafting
    handleDiagramCrafting((DiagramCraftingBench) benchConfig);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new DiagramCraftingBench(). Doing so bypasses the asset loading pipeline, resulting in an unconfigured and useless object. All instances must be created by the engine's codec system.
- **Runtime Modification:** Do not modify the state of a retrieved DiagramCraftingBench instance. These objects are shared across the entire server; runtime changes will cause global state corruption.

## Data Pipeline
The DiagramCraftingBench is the output of a data deserialization pipeline. It does not process data itself; it *is* the data.

> Flow:
> Game Asset File (e.g., `my_mod:diagram_table.json`) -> AssetManager -> Hytale Codec System -> **DiagramCraftingBench.CODEC** -> Instantiated **DiagramCraftingBench** Object -> Stored in BlockType Component Registry

