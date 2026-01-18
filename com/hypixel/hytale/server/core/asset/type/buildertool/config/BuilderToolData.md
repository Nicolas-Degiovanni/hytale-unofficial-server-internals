---
description: Architectural reference for BuilderToolData
---

# BuilderToolData

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config
**Type:** Data Model / DTO

## Definition
```java
// Signature
public class BuilderToolData implements NetworkSerializable<ItemBuilderToolData> {
```

## Architecture & Concepts
The BuilderToolData class is a server-side data model that represents the complete configuration for an in-game builder tool item. It is a passive data structure, not an active engine component. Its primary architectural role is to serve as an intermediate representation between persistent storage (configuration files) and the network protocol.

This class encapsulates the user interface layout (the `ui` field) and the set of available sub-tools (the `tools` field). The static `CODEC` field is the most critical feature, defining the contract for how this object is deserialized from asset files. By implementing the NetworkSerializable interface, this class explicitly signals its responsibility to be converted into a network-safe packet format, specifically the `ItemBuilderToolData` packet, for transmission to the client.

In essence, BuilderToolData is the canonical server-side representation of a tool's properties before they are packaged for network transit.

## Lifecycle & Ownership
- **Creation:** Instances are primarily created by the Hytale `Codec` framework during the server's asset loading phase. The framework reads a configuration file (e.g., a JSON asset) and uses the static `CODEC` definition to instantiate and populate a BuilderToolData object. Manual instantiation via `new BuilderToolData()` is possible but generally reserved for testing or programmatic tool generation. A static `DEFAULT` instance exists for fallback scenarios.

- **Scope:** Transient and short-lived. An instance typically exists only within the scope of an asset loading operation or a player-specific logic flow. It is not a long-lived service and is not registered in any central registry.

- **Destruction:** The object is eligible for garbage collection as soon as all references to it are dropped. This typically occurs after its data has been used to create a network packet via the `toPacket` method or when the associated configuration is reloaded.

## Internal State & Concurrency
- **State:** Mutable. The internal `ui` and `tools` arrays are populated after construction, either by the constructor or directly by the `CODEC` during deserialization. The object does not contain any other state or cache.

- **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded access, typically on the server's main thread or a dedicated asset-loading thread. Concurrent modification of the internal arrays will lead to race conditions and undefined behavior. All operations on an instance should be synchronized externally if multi-threaded access is unavoidable.

## API Surface
The public API is minimal, focusing on data retrieval and network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | ItemBuilderToolData | O(N) | Converts the data model into its corresponding network packet. N is the number of tools. This is the primary operation for this class. |
| getUi() | String[] | O(1) | Returns a direct reference to the internal UI definition array. |
| getTools() | BuilderTool[] | O(1) | Returns a direct reference to the internal tool definition array. |

## Integration Patterns

### Standard Usage
The intended use is for the server's asset system to deserialize this object from a configuration file. Game logic then retrieves this configured object and uses it to inform the client about the tool's state, typically upon player login or when a tool is equipped.

```java
// Example: Hypothetical asset loading and packet creation
// NOTE: Developers do not typically call the CODEC directly.

// 1. The system loads the configuration into a BuilderToolData instance.
BuilderToolData toolConfig = AssetSystem.load("my_awesome_tool.json", BuilderToolData.CODEC);

// 2. Game logic converts it to a packet to send to a player.
ItemBuilderToolData packet = toolConfig.toPacket();
player.getConnection().send(packet);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation After Load:** Do not modify the internal arrays returned by `getUi` or `getTools` after the object has been loaded from configuration. This creates a state divergence between the persistent asset and the in-memory representation, which can cause severe debugging challenges. Treat the object as immutable after creation.

- **Shared Instances:** Do not share a single BuilderToolData instance across multiple players if their tool states can diverge. Each player-specific state should be derived from a fresh copy or a new packet.

## Data Pipeline
This class is a key link in the pipeline that transforms a static server asset into a dynamic update for the game client.

> Flow:
> Server Asset File (JSON) -> Hytale Codec Deserializer -> **BuilderToolData Instance** -> `toPacket()` Invocation -> `ItemBuilderToolData` Packet -> Server Network Layer -> Client Game Engine

