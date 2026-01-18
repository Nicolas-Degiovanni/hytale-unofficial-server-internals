---
description: Architectural reference for StringArg
---

# StringArg

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config.args
**Type:** Transient

## Definition
```java
// Signature
public class StringArg extends ToolArg<String> {
```

## Architecture & Concepts
The StringArg class is a concrete implementation of the generic ToolArg abstraction. Its primary role is to represent a string-based argument within the configuration of a server-side Builder Tool. This class is a fundamental component of the data-driven tool system, acting as a bridge between static asset definitions and the live network protocol.

Architecturally, StringArg is a self-describing data model. The static CODEC field exposes its structure to the Hytale serialization framework, allowing the server to automatically deserialize tool configurations from asset files into strongly-typed Java objects.

Furthermore, it functions as a factory for its corresponding network packet, BuilderToolStringArg. This design encapsulates the logic for converting a high-level configuration concept into a low-level, transmissible data structure, decoupling the tool definition system from the raw network layer.

### Lifecycle & Ownership
- **Creation:** StringArg instances are primarily created by the BuilderCodec during the server's asset loading phase. When the server parses a builder tool definition file (e.g., a JSON or HOCON file), the codec instantiates StringArg to represent any arguments of this type. Manual instantiation via `new StringArg("value")` is possible but typically reserved for programmatic tool generation.
- **Scope:** The lifetime of a StringArg object is bound to its parent Builder Tool configuration. It is a short-lived, transient object that exists only while the tool's definition is held in memory.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for collection once its parent configuration is unloaded or no longer referenced. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The core state is the `value` field, inherited from the parent ToolArg class. This state is **mutable**. The default constructor creates an empty object, and the `value` is subsequently populated by the BuilderCodec during deserialization.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization primitives. It is designed to be instantiated and configured within a single-threaded context, such as the server's main startup thread during asset loading.

**WARNING:** Modifying a StringArg object from multiple threads after its initial creation is an unsupported operation and will lead to unpredictable behavior and race conditions. Treat instances as effectively immutable after they have been deserialized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromString(String str) | String | O(1) | Fulfills the ToolArg contract for string-based conversion. Returns the input string directly. |
| toStringArgPacket() | BuilderToolStringArg | O(1) | Creates a new, specialized network packet containing the argument's value. |
| setupPacket(BuilderToolArg packet) | void | O(1) | Configures a generic BuilderToolArg packet, setting its type discriminator and attaching the specific string packet payload. |

## Integration Patterns

### Standard Usage
The intended use of StringArg is declarative. A developer defines a tool's arguments within a static asset file. The server's asset pipeline uses the provided CODEC to automatically instantiate and populate the object. The system later uses this object to serialize the tool's definition into network packets for the client.

```java
// System-level code (conceptual)
// The system retrieves the pre-configured ToolArg from a loaded asset.
// This instance was created by the BuilderCodec, not manually.
ToolArg<?> arg = loadedTool.getArgument("message");

// The system prepares a generic packet
BuilderToolArg packet = new BuilderToolArg();

// The arg object itself populates the packet with its specific type and data
arg.setupPacket(packet);

// The packet is now ready for network serialization
networkManager.send(packet);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not modify the `value` of a StringArg after it has been initialized by the asset loader. The object should be treated as immutable post-creation to ensure predictable network serialization.
- **Cross-Purpose Use:** Do not use this class for general-purpose string storage. It is tightly coupled to the Builder Tool system and its network protocol, and its use outside this context is not supported.

## Data Pipeline
StringArg serves as a critical link in the data flow from server configuration to client-side tool representation.

> Flow:
> Builder Tool Asset File -> AssetManager -> **BuilderCodec (using StringArg.CODEC)** -> In-Memory **StringArg** Instance -> `setupPacket` call -> BuilderToolStringArg Packet -> Network Layer -> Client

