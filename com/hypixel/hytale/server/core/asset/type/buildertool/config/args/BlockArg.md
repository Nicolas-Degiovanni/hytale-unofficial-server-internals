---
description: Architectural reference for BlockArg
---

# BlockArg

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config.args
**Type:** Transient Data Model

## Definition
```java
// Signature
public class BlockArg extends ToolArg<BlockPattern> {
```

## Architecture & Concepts
The BlockArg class is a specialized data model representing a configurable argument for a server-side Builder Tool. Its primary function is to define an argument that accepts a Hytale block or a complex block pattern as its value. For example, a "fill" tool might use a BlockArg to specify what material to fill with.

This class acts as a critical bridge between static game assets and the live network protocol. It is designed to be deserialized directly from configuration files via its static **CODEC** field. This allows designers to define the behavior and constraints of builder tools in simple text files without modifying server code.

Key architectural features include:
- **Declarative Initialization:** The static **BuilderCodec** enables the server's asset loading system to construct and populate BlockArg instances automatically from asset definitions.
- **Type Specialization:** By extending the generic ToolArg, it provides a type-safe implementation for handling values of type BlockPattern.
- **Protocol Translation:** It contains the logic (**toBlockArgPacket**) to convert its server-side configuration state into a specific network packet, BuilderToolBlockArg, for transmission to the game client. This decouples the server's internal configuration from the client-facing network contract.

## Lifecycle & Ownership
- **Creation:** BlockArg instances are not intended for manual instantiation. They are created exclusively by the **BuilderCodec** system during the server's asset loading phase at startup. The codec reads a builder tool definition file and constructs the entire object graph, including any BlockArg members.
- **Scope:** An instance of BlockArg has a lifetime equivalent to the server session. It is loaded once and persists in memory as part of its parent builder tool's configuration. It should be treated as a read-only data object after initial loading.
- **Destruction:** The object is marked for garbage collection when the server shuts down or when the asset registry is cleared during a hot-reload process.

## Internal State & Concurrency
- **State:** The internal state, consisting of the inherited *value* (a BlockPattern) and the *allowPattern* boolean, is mutable only during the deserialization process managed by the **CODEC**. After asset loading is complete, the state is considered frozen and should not be modified.
- **Thread Safety:** This class is **not thread-safe** for mutation. It lacks any internal locking mechanisms. However, it is safe for concurrent reads from multiple threads (e.g., command handlers for different players) under the strict assumption that its state is not modified after the initial server bootstrap.

**WARNING:** Any runtime modification of a BlockArg instance after it has been loaded will lead to unpredictable and inconsistent behavior across the server.

## API Surface
The public API is focused on translation and data access, not state mutation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCodec() | Codec<BlockPattern> | O(1) | Returns the codec for the underlying BlockPattern type. |
| fromString(String str) | BlockPattern | O(N) | Parses a string representation into a BlockPattern object. |
| toBlockArgPacket() | BuilderToolBlockArg | O(N) | Creates a new network packet DTO from this object's state. |

## Integration Patterns

### Standard Usage
BlockArg is not used directly but is accessed through a parent configuration object that represents the entire builder tool. Game logic retrieves the pre-configured argument to inform its behavior.

```java
// Assume 'toolConfig' is a pre-loaded asset for a specific builder tool
// This toolConfig would contain a list or map of ToolArg instances

// Retrieve the specific block argument by its configured name
BlockArg materialArg = (BlockArg) toolConfig.getArgument("material");

// Use its properties to create a network packet to send to the client
BuilderToolArg packet = new BuilderToolArg();
materialArg.setupPacket(packet); // Populates the packet with block-specific data
networkManager.sendPacket(player, packet);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockArg()`. The object's state will be incomplete and invalid. The asset system and its **CODEC** are solely responsible for its creation.
- **Runtime State Mutation:** Modifying the fields of a cached BlockArg instance after server startup is a severe anti-pattern. This breaks the "configuration as code" paradigm and can cause race conditions and inconsistent tool behavior for different players.

```java
// DO NOT DO THIS
BlockArg arg = toolConfig.getArgument("material");
arg.allowPattern = false; // DANGEROUS: This mutates shared, cached configuration
```

## Data Pipeline
BlockArg serves as a transformation point, converting static file data into in-memory configuration and finally into network packets.

> Flow:
> Builder Tool Asset File (e.g., JSON) -> Server Asset Loader -> **BlockArg CODEC** -> In-Memory **BlockArg** Instance -> `toBlockArgPacket()` -> Network Packet -> Client UI

