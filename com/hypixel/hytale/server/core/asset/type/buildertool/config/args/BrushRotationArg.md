---
description: Architectural reference for BrushRotationArg
---

# BrushRotationArg

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config.args
**Type:** Model / Data Transfer Object

## Definition
```java
// Signature
public class BrushRotationArg extends ToolArg<Rotation> {
```

## Architecture & Concepts
The BrushRotationArg class is a strongly-typed data model that represents a single, configurable rotation argument for a server-side Builder Tool. It serves as a critical bridge between static game assets and the runtime network protocol.

Its primary architectural purpose is to enable data-driven design for in-game tools. Through its static **CODEC** field, game designers can define a tool's rotation behavior in an external configuration file (e.g., JSON or HOCON). The Hytale asset loading system uses this codec to deserialize the configuration into a memory-resident BrushRotationArg object.

At runtime, when a player uses the configured tool, this object is responsible for translating its internal **Rotation** enum state into a specialized network packet, **BuilderToolRotationArg**, for transmission to the client. It encapsulates the schema, validation, and serialization logic for one specific type of tool parameter.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the **BuilderCodec** deserialization system when a server boots up and loads builder tool asset files. Direct manual instantiation is rare and reserved for procedurally generated or hard-coded tools.
- **Scope:** The object's lifetime is bound to its parent tool configuration asset. It persists in memory as long as the tool definition is loaded on the server. It is effectively an immutable value object post-initialization.
- **Destruction:** The object is marked for garbage collection when the server unloads the corresponding asset pack or shuts down. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** The core state is a single, mutable field named **value** (inherited from ToolArg) of type **Rotation**. This field is populated during deserialization and is considered final thereafter. The class itself is stateless, but the instances it creates are stateful.
- **Thread Safety:** This class is **not thread-safe**. Instances are designed to be created, owned, and read by a single thread, typically the main server thread. All asset loading and subsequent game logic that interacts with these objects must be synchronized or confined to a single thread to prevent data corruption.

## API Surface
The public API is minimal, focusing on translation and packet creation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromString(String str) | Rotation | O(1) | Deserializes a string into a Rotation enum. Throws ToolArgException on invalid input. |
| toRotationArgPacket() | BuilderToolRotationArg | O(1) | Creates a new, specialized network packet from the internal state. |
| setupPacket(BuilderToolArg packet) | void | O(1) | Configures a generic network packet with the specific rotation data. This is the primary integration point for the network layer. |

## Integration Patterns

### Standard Usage
Developers typically do not create or manage BrushRotationArg instances directly. Instead, they interact with them after they have been loaded as part of a larger tool definition. The primary pattern is retrieving the argument from a tool and using it to construct network packets.

```java
// This is system-level code, not typical gameplay logic.
// An instance of BrushRotationArg is retrieved from a loaded tool asset.

// 1. A generic packet is created.
BuilderToolArg genericPacket = new BuilderToolArg();

// 2. The BrushRotationArg instance populates the packet.
// The 'arg' variable is assumed to be a loaded BrushRotationArg.
arg.setupPacket(genericPacket);

// 3. The fully-configured packet is sent over the network.
// networkSubsystem.sendPacketToClient(player, genericPacket);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation After Load:** Do not modify the internal **value** of a BrushRotationArg after it has been loaded from configuration. This can lead to inconsistent server state, as the behavior will deviate from the asset definition.
- **Cross-Thread Access:** Never read or write a BrushRotationArg instance from a worker thread without external locking. These objects are not designed for concurrent access.
- **Manual Instantiation:** Avoid `new BrushRotationArg()` in gameplay logic. This bypasses the data-driven asset system and hard-codes tool behavior, defeating the purpose of the architecture.

## Data Pipeline
The BrushRotationArg acts as a transformation stage in the data flow from game assets to the client.

> Flow:
> Builder Tool Asset File (JSON/HOCON) -> Hytale Codec System -> **BrushRotationArg Instance** -> `setupPacket()` Method -> BuilderToolArg Network Packet -> Client Game Engine

