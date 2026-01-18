---
description: Architectural reference for JsonUpdateType
---

# JsonUpdateType

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Enum Constant

## Definition
```java
// Signature
public enum JsonUpdateType {
```

## Architecture & Concepts
JsonUpdateType is a type-safe enumeration that defines the set of valid modification operations for JSON data structures within the asset editor protocol. It serves as a contract between the client and server, ensuring that only supported update types are processed.

This enum is a critical component for network serialization. Instead of transmitting verbose string commands like "SetProperty", the engine serializes the command into a compact integer via the *getValue* method. The receiving end then deserializes this integer back into its corresponding enum constant using the static *fromValue* factory method. This pattern significantly reduces packet size and deserialization complexity.

The presence of a static *VALUES* array is a micro-optimization to prevent repeated allocation of a new array every time the *values()* method is called, which is common in high-performance network code.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine during class loading. They are compile-time constants, not runtime objects in the traditional sense.
- **Scope:** As static final instances, they exist for the entire lifetime of the application. Their scope is global.
- **Destruction:** They are garbage collected only when the defining class loader is unloaded, which typically happens at JVM shutdown.

## Internal State & Concurrency
- **State:** JsonUpdateType is immutable. Its internal state, the integer *value*, is final and assigned at compile time.
- **Thread Safety:** This enum is intrinsically thread-safe. Its immutability guarantees that it can be safely accessed and shared across any number of threads without synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the enum constant for network serialization. |
| fromValue(int value) | JsonUpdateType | O(1) | Deserializes an integer into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the serialization and deserialization of asset editor network packets.

```java
// Example: Serializing an update command
int commandId = JsonUpdateType.SetProperty.getValue();
// ... write commandId to network buffer ...

// Example: Deserializing an incoming command
int receivedId = // ... read from network buffer ...
try {
    JsonUpdateType command = JsonUpdateType.fromValue(receivedId);
    // ... process the command ...
} catch (ProtocolException e) {
    // Handle malicious or corrupted packet
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Exceptions:** Failure to wrap calls to *fromValue* in a try-catch block can crash the network processing thread if a malformed or malicious packet with an invalid integer value is received.
- **Using ordinal():** Do not rely on the built-in *ordinal()* method for serialization. The explicit *value* field is used to guarantee that the network contract remains stable even if the declaration order of the enum constants changes in the future. This code correctly uses a dedicated value.

## Data Pipeline
JsonUpdateType represents a command that dictates how a JSON modification packet should be processed. It is data, not an active component, within the asset editor's data flow.

> Flow:
> User Action in Editor -> **JsonUpdateType** selected -> Packet Serializer (uses *getValue*) -> Network -> Packet Deserializer (uses *fromValue*) -> Asset Processor -> JSON Asset Mutated

