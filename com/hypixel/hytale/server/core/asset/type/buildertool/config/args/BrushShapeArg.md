---
description: Architectural reference for BrushShapeArg
---

# BrushShapeArg

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config.args
**Type:** Transient Data Model

## Definition
```java
// Signature
public class BrushShapeArg extends ToolArg<BrushShape> {
```

## Architecture & Concepts
The BrushShapeArg class is a concrete implementation of the generic ToolArg, specialized for handling builder tool arguments of the type BrushShape. It serves as a critical data model that bridges two distinct engine systems: the server-side asset configuration system and the client-server network protocol.

As part of a larger polymorphic design, BrushShapeArg allows the builder tool system to manage various argument types (e.g., size, color, shape) through the common ToolArg interface. Its primary responsibilities are:

1.  **Data Representation:** To hold a specific BrushShape enum value that defines a tool's behavior.
2.  **Serialization Contract:** To define, via its static CODEC field, how it is deserialized from Hytale's configuration files (e.g., JSON or HOCON) into a live Java object.
3.  **Packet Transformation:** To convert its internal state into a network-ready packet format (BuilderToolArg) for synchronization with the game client.

This class encapsulates the logic for a single, specific type of tool parameter, promoting a clean separation of concerns within the builder tool framework.

### Lifecycle & Ownership
-   **Creation:** BrushShapeArg instances are primarily created by the Hytale codec system during server boot or asset reloading. The static `CODEC` field is used by a parent `BuilderCodec` to deserialize a tool's configuration from a file and instantiate this class. Manual instantiation via `new BrushShapeArg(value)` is possible but less common.
-   **Scope:** The lifetime of a BrushShapeArg instance is tied to its parent tool configuration object. It is a value object and does not persist beyond the lifetime of the tool definition it belongs to.
-   **Destruction:** The object is eligible for garbage collection as soon as the containing tool configuration is unloaded or no longer referenced. It requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The class is a simple, mutable data container. Its primary state, the BrushShape enum, is stored in the `value` field inherited from the parent ToolArg class. The static CODEC and BRUSH_SHAPE_CODEC fields are immutable and shared across all instances.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be configured during the single-threaded asset loading phase and subsequently treated as a read-only data object. Modifying its state from multiple threads will lead to unpredictable behavior and race conditions. All network packet generation should occur in a thread-safe context, such as within a specific game loop or network thread that has exclusive access to the object at that time.

## API Surface
The public API is focused on data conversion and network packet preparation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromString(str) | BrushShape | O(1) | **DEPRECATED-BEHAVIOR**. Creates a BrushShape enum from its string name. Prone to runtime exceptions if the string is invalid. |
| toBrushShapeArgPacket() | BuilderToolBrushShapeArg | O(1) | Creates a new, specialized network packet containing the BrushShape value. |
| setupPacket(packet) | void | O(1) | Configures a generic BuilderToolArg packet with the specific data for this argument type. This is the primary integration point with the network layer. |

## Integration Patterns

### Standard Usage
The most common interaction pattern involves the engine using a tool's list of ToolArg instances to populate a network packet that describes the tool to the client. The polymorphic `setupPacket` method is called on each argument to fill out the details.

```java
// Hypothetical tool preparing its arguments for network sync
BuilderTool someTool = ...;
BuilderToolArg genericPacket = new BuilderToolArg();

// The engine finds the correct BrushShapeArg instance for the tool
BrushShapeArg shapeArg = someTool.getArgument(BrushShapeArg.class);

// The argument itself populates the generic packet
// This call sets the argType and attaches the specific brushShapeArg data
shapeArg.setupPacket(genericPacket);

// The genericPacket is now ready to be sent
networkManager.send(genericPacket);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation After Packet Creation:** Modifying the internal `value` of a BrushShapeArg after it has been used to configure a packet or tool can cause a state desynchronization between the server's understanding of the tool and what the client has been told. Treat these instances as immutable after initial loading.
-   **Manual String Parsing:** Relying on the `fromString` method for user input or other dynamic sources is dangerous. It uses `Enum.valueOf`, which will throw an `IllegalArgumentException` for invalid input, potentially crashing the server thread if not handled carefully. Always use the provided codec system for deserialization.

## Data Pipeline
BrushShapeArg acts as a transformation stage in the data flow from static configuration files to live network packets.

> Flow:
> Asset File (e.g., tool.json) -> Hytale Codec Engine -> **BrushShapeArg Instance** -> `setupPacket()` -> BuilderToolArg (Network Packet) -> Network Serialization Layer -> Client

---

