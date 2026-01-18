---
description: Architectural reference for BrushData
---

# BrushData

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config
**Type:** Transient

## Definition
```java
// Signature
public class BrushData implements NetworkSerializable<BuilderToolBrushData> {
```

## Architecture & Concepts
The BrushData class is a server-side data model that represents the complete configuration of a builder tool's brush. It serves as a mutable container for all properties that define a brush's behavior, such as its shape, size, material, and complex masking rules.

Architecturally, this class is a composite object, not a simple Plain Old Java Object. Each property is encapsulated within a specific **ToolArg** wrapper class (e.g., IntArg, BlockArg, MaskArg). This design pattern delegates concerns like validation, range constraints, default values, and string-based parsing to the individual field types, keeping the BrushData class itself clean and focused on aggregation.

The class is fundamentally integrated with the engine's serialization system through its static **CODEC** field. This `BuilderCodec` provides a declarative schema for deserializing BrushData from asset configuration files. It maps string keys from the asset file (e.g., "Width", "Shape") directly to the corresponding fields in a new BrushData instance, forming the backbone of the asset loading pipeline for builder tools.

A critical design choice is the inclusion of the nested **BrushData.Values** class.
*   **BrushData** is the *mutable configuration object*. It is designed to be modified by UI events or commands via its `updateArgValue` method.
*   **BrushData.Values** is an *immutable snapshot* of the raw data within a BrushData instance. It is created on-demand to provide a stable, thread-safe representation of the brush's state for use in performance-critical world modification logic. This separation prevents the core brush application algorithms from dealing with the complexity of the ToolArg wrappers and protects against mid-operation state changes.

Finally, its implementation of the NetworkSerializable interface marks it as a key component in the client-server communication protocol. It can be converted into a network-safe packet object for synchronizing a player's brush settings with the client.

### Lifecycle & Ownership
- **Creation:** BrushData instances are primarily created by the engine's asset loading system using the static CODEC. They are also instantiated programmatically to represent a player's active or default tool configuration. The static DEFAULT instance serves as a fallback or initial state.
- **Scope:** The lifetime of a BrushData instance is typically bound to a player's session with a specific builder tool. It is not a global singleton and persists as long as the tool configuration is active.
- **Destruction:** The object is eligible for garbage collection when it is no longer referenced, for example, when a player changes their tool configuration, disconnects from the server, or the associated asset is unloaded.

## Internal State & Concurrency
- **State:** The state of a BrushData object is **highly mutable**. Its primary function is to hold a configuration that can be changed at any time by the player. All fields representing brush properties can be updated.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. All method calls that read or modify its state must be performed on the main server thread.

**WARNING:** Accessing or modifying a BrushData instance from worker threads will lead to race conditions, data corruption, and server instability. For multi-threaded operations, create an immutable `BrushData.Values` snapshot on the main thread and pass the snapshot to the worker thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| updateArgValue(brush, id, value) | void | O(1) | Mutates a single property based on its string key. Throws ToolArgException on invalid input. |
| toPacket() | BuilderToolBrushData | O(N) | Serializes the current state into a network packet. N is the count of favorite materials and mask commands. |
| getWidth() | IntArg | O(1) | Returns the wrapper object for the width property. |
| getShape() | BrushShapeArg | O(1) | Returns the wrapper object for the shape property. |

## Integration Patterns

### Standard Usage
The correct pattern is to treat BrushData as a mutable configuration model and BrushData.Values as an immutable parameter for game logic.

```java
// 1. Obtain the active brush configuration for a player
BrushData activeBrushConfig = player.getBuilderTool().getBrushData();

// 2. A UI event or command triggers a change
try {
    activeBrushConfig.updateArgValue(values, "Width", "10");
} catch (ToolArgException e) {
    player.sendMessage(e.getMessage());
}

// 3. When the brush is used, create an immutable snapshot for processing
BrushData.Values brushSnapshot = new BrushData.Values(activeBrushConfig);

// 4. Pass the clean, immutable snapshot to the world modification system
world.applyBrush(player.getPosition(), brushSnapshot);
```

### Anti-Patterns (Do NOT do this)
- **Cross-Thread Modification:** Never call `updateArgValue` or modify a BrushData instance from a separate thread. Always dispatch such operations to the main server thread.
- **Reusing Snapshots:** Do not cache and reuse a `BrushData.Values` object across multiple brush applications. A new snapshot must be created before each use to ensure it reflects the latest configuration.
- **Direct Field Access:** Avoid directly accessing the protected ToolArg fields. While possible, this bypasses the intended mutation pathway through `updateArgValue` and can lead to inconsistent state.

## Data Pipeline
The BrushData class sits at the junction of asset configuration, runtime state management, and network synchronization.

> **Asset Loading Flow:**
> Builder Tool Asset File (JSON) -> Engine Asset Loader -> **BuilderCodec** -> **BrushData** Instance

> **Brush Application Flow:**
> Player Input -> Command/Event Handler -> `updateArgValue` on **BrushData** -> Player uses tool -> `new BrushData.Values()` -> World Modification System

> **Network Synchronization Flow:**
> Server-side **BrushData** state change -> `toPacket()` -> `BuilderToolBrushData` Packet -> Network Layer -> Client UI Update

