---
description: Architectural reference for ToolArg
---

# ToolArg

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config.args
**Type:** Abstract Data Type

## Definition
```java
// Signature
public abstract class ToolArg<T> implements NetworkSerializable<BuilderToolArg> {
```

## Architecture & Concepts
The ToolArg class is an abstract base for defining arguments used by server-side builder tools. It establishes a formal contract for how tool parameters are defined, serialized, deserialized, and transmitted over the network. This class is a cornerstone of Hytale's data-driven builder tool system, enabling developers to create new, complex tools by composing arguments without modifying the core engine.

Its primary architectural roles are:
- **Polymorphic Contract:** It defines the essential behaviors that all specific argument types, such as a Vector3, a Block ID, or a String, must implement.
- **Serialization Gateway:** Through its static CODEC field and the abstract getCodec method, it integrates directly with the engine's powerful codec system. The CodecMapCodec is specifically used to handle the polymorphism, allowing the system to dynamically serialize and deserialize the correct subclass based on a "Type" identifier in the data.
- **Network Marshalling:** By implementing NetworkSerializable, it serves as the bridge between the high-level tool configuration and the low-level network packet, BuilderToolArg. It is responsible for marshalling its own state into a network-ready format.

This class embodies the Template Method design pattern via the toPacket and setupPacket methods. The public toPacket method defines the skeleton of the packet creation algorithm, while delegating the type-specific details to the abstract setupPacket method, which must be implemented by subclasses.

## Lifecycle & Ownership
- **Creation:** Instances of ToolArg are never created directly via a constructor. They are instantiated by the Hytale codec system during the deserialization of asset files (e.g., JSON definitions for a builder tool) or when parsing network data. The static CodecMapCodec is the primary factory for all ToolArg objects.
- **Scope:** The lifetime of a ToolArg instance is bound to the lifecycle of its owning builder tool configuration. It is loaded into memory when the tool asset is loaded and persists as long as that tool is available on the server.
- **Destruction:** The object is eligible for garbage collection when the server unloads the associated tool assets or during a server shutdown sequence. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** Mutable. A ToolArg instance holds two primary pieces of state: a boolean flag, *required*, and a generic field, *value*. The value is populated during deserialization from a configuration file or after parsing from a user-provided command string.
- **Thread Safety:** **Not thread-safe.** This class is designed as a simple data container and is not intended for concurrent access. All operations on a ToolArg instance are expected to occur on the main server thread within the context of the game loop or a specific command-handling routine. No internal locking mechanisms are present.

**WARNING:** Modifying the state of a ToolArg from an asynchronous task or a different thread will lead to race conditions and unpredictable server behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | T | O(1) | Returns the parsed and validated value of the argument. |
| isRequired() | boolean | O(1) | Returns true if this argument must be provided for the tool to execute. |
| getCodec() | Codec<T> | O(1) | **Abstract.** Subclasses must return the specific codec for their generic type T. |
| fromString(String) | T | O(N) | **Abstract.** Subclasses must implement logic to parse a raw string into the typed value T. Throws ToolArgException on failure. |
| setupPacket(BuilderToolArg) | void | O(1) | **Protected Abstract.** Subclasses must implement this to populate the type-specific fields of a network packet. |
| toPacket() | BuilderToolArg | O(1) | Creates and returns a network-ready packet representing this argument's state. |

## Integration Patterns

### Standard Usage
A developer never uses ToolArg directly. Instead, they create a concrete implementation for a new data type and register it with the central codec.

```java
// 1. Define a concrete argument type
public class StringToolArg extends ToolArg<String> {
    public static final Codec<StringToolArg> CODEC = ... // Codec definition

    @Override
    public Codec<String> getCodec() {
        return Codec.STRING;
    }

    @Override
    public String fromString(String input) {
        return input; // Simple pass-through
    }

    @Override
    protected void setupPacket(BuilderToolArg packet) {
        packet.value = new StringArgumentValue(this.value);
    }
}

// 2. Register the new type with the central codec map
// This typically happens during a static initialization block or engine bootstrap.
ToolArg.CODEC.register("String", StringToolArg.CODEC);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to call `new ToolArg()`. This is an abstract class and will result in a compilation error. All instances must be created via the codec system.
- **Ignoring Immutability:** While the class is mutable, treat instances as immutable after they have been deserialized from assets. Modifying the state of a shared ToolArg instance at runtime can lead to inconsistent behavior across different systems that reference it.
- **Complex Logic in fromString:** The fromString method should be reserved for simple parsing and validation. Complex game logic, such as querying the world state, should not be performed here as it can block the command processing thread.

## Data Pipeline
The ToolArg class is a critical component in several data flows within the server.

> **Asset Deserialization Flow:**
> JSON/HOCON File on Disk -> AssetLoader -> **CodecMapCodec** -> Instantiates concrete **ToolArg** subclass -> Stored in BuilderTool configuration

> **Network Serialization Flow:**
> Server-side **ToolArg** instance -> `toPacket()` -> `BuilderToolArg` (Packet) -> Protocol Layer -> Network Buffer -> Client

> **Command Parsing Flow:**
> Player Chat Input (e.g., "/tool place stone") -> Command Parser -> `fromString()` on corresponding **ToolArg** -> Populates `value` field -> Tool Execution Logic

