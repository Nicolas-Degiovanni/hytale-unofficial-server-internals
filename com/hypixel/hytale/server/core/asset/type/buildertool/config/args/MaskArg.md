---
description: Architectural reference for MaskArg
---

# MaskArg

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config.args
**Type:** Transient

## Definition
```java
// Signature
public class MaskArg extends ToolArg<BlockMask> {
```

## Architecture & Concepts
The MaskArg class is a specialized Data Transfer Object (DTO) that represents a block selection rule for server-side Builder Tools. It serves as a strongly-typed container for a BlockMask, which defines complex criteria for selecting blocks in the world (e.g., "all stone blocks" or "any block with a specific property").

This class is a critical component of the server's tooling and prefab system. Its primary architectural function is to act as a bridge between high-level asset definitions (likely HOCON or JSON files defining a tool's behavior) and the low-level network protocol. The static CODEC field enables the Hytale serialization framework to automatically deserialize tool configurations from asset files into in-memory MaskArg objects.

When a tool is used, the MaskArg instance is then responsible for serializing its internal BlockMask state into a specific network packet, BuilderToolMaskArg, for transmission to the client. This ensures a consistent and type-safe data flow from static configuration to runtime network communication.

## Lifecycle & Ownership
-   **Creation:** MaskArg instances are almost exclusively created by the Hytale serialization framework during the asset loading phase. When the server parses a Builder Tool asset file, the defined arguments are deserialized into corresponding ToolArg subclasses, including MaskArg. Manual instantiation is rare and typically confined to unit tests or dynamic tool generation.
-   **Scope:** The lifetime of a MaskArg instance is bound to its parent tool configuration object. It is a short-lived, immutable-by-convention object that persists only as long as the tool definition is held in memory. It is not a singleton or a long-lived service.
-   **Destruction:** The object is marked for garbage collection when its parent tool configuration is unloaded or no longer referenced.

## Internal State & Concurrency
-   **State:** The class is **Mutable**, though it should be treated as immutable after initial deserialization. Its core state consists of a BlockMask object and a boolean *required* flag, both inherited from ToolArg. The public CODEC directly mutates the internal *value* field during object creation, a common pattern in Hytale's serialization system.
-   **Thread Safety:** This class is **not thread-safe**. It is a simple data container with no internal locking or synchronization. All operations on a MaskArg instance are expected to occur on the main server thread or within a single-threaded asset loading context. Concurrent access from multiple threads will result in undefined behavior and is strictly unsupported.

## API Surface
The public API is focused on serialization and network packet creation, reinforcing its role as a data marshalling component.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromString(String str) | BlockMask | O(N) | **Deserialization Helper.** Parses a string representation into a BlockMask object. Throws exceptions on malformed input. |
| toMaskArgPacket() | BuilderToolMaskArg | O(N) | **Packet Factory.** Creates a specialized, network-ready packet containing the BlockMask data. |
| setupPacket(BuilderToolArg packet) | void | O(N) | **Packet Configuration.** Populates a generic BuilderToolArg packet with the specific type and payload for a mask argument. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in typical game logic. It is part of the underlying tool configuration framework. The system interacts with it after it has been deserialized from an asset file.

```java
// This class is deserialized from an asset, not manually constructed.
// The following is a conceptual example of how the system uses a pre-loaded MaskArg.

// 1. A tool's configuration is loaded from assets.
//    The 'replaceToolConfig' object now contains a MaskArg instance.
BuilderToolConfig replaceToolConfig = assetManager.load("mytools:replace_stone");

// 2. The system retrieves the strongly-typed argument.
MaskArg sourceMask = replaceToolConfig.getArgument("source_blocks", MaskArg.class);

// 3. When the tool is executed, the argument is used to create a network packet.
BuilderToolArg packet = new BuilderToolArg();
sourceMask.setupPacket(packet);

// 4. The fully configured packet is sent to the client.
networkManager.sendPacket(player, packet);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation:** Do not modify the internal *value* or *required* fields after the object has been initialized by the asset loader. This violates the "immutable-by-convention" design and can lead to inconsistent state between the server and client.
-   **Manual Serialization:** Do not attempt to serialize this object using custom logic. Always rely on the provided static CODEC to ensure forward and backward compatibility with the engine's serialization format.
-   **Direct Instantiation:** Avoid using `new MaskArg()`. The object's state is meant to be defined declaratively in asset files, not programmatically constructed during runtime.

## Data Pipeline
The MaskArg class is a key stage in the data flow from static tool definition to client-side execution.

> Flow:
> Builder Tool Asset File (HOCON/JSON) -> Asset Loader using **MaskArg.CODEC** -> In-memory **MaskArg** instance -> Server Tooling Logic -> `setupPacket()` -> `BuilderToolArg` Network Packet -> Network Layer -> Client Renderer/Logic

