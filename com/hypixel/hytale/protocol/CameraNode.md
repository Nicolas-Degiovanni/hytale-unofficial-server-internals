---
description: Architectural reference for CameraNode
---

# CameraNode

**Package:** com.hypixel.hytale.protocol
**Type:** Enum / Utility

## Definition
```java
// Signature
public enum CameraNode {
   None(0),
   Head(1),
   LShoulder(2),
   RShoulder(3),
   Belly(4);
```

## Architecture & Concepts
The CameraNode enum provides a type-safe, compile-time constant representation for camera attachment points on an entity. It is a fundamental component of the protocol layer, designed to translate low-level integer identifiers—received from the network or read from game files—into a high-level, self-documenting object.

Its primary architectural function is to act as a data integrity and validation boundary. By encapsulating the mapping between integers and named constants, it eliminates the use of "magic numbers" throughout the rendering and gameplay systems. The static factory method, fromValue, serves as a strict deserialization gate, throwing a ProtocolException for any undefined integer value, thereby preventing invalid state from propagating deeper into the engine.

## Lifecycle & Ownership
- **Creation:** Instantiated by the Java Virtual Machine during class loading. The set of instances (None, Head, etc.) is fixed and created only once.
- **Scope:** Application-wide singleton scope. The enum constants persist for the entire lifetime of the application.
- **Destruction:** Managed by the JVM. The constants are eligible for garbage collection only when the CameraNode class is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final, private integer value. The class also maintains a static final array of its constants (VALUES) for O(1) lookups, which is populated once during class initialization.
- **Thread Safety:** Inherently thread-safe. As a Java enum, all instances are constants initialized securely by the class loader. No external synchronization is required for access from any thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the underlying integer representation for serialization. |
| fromValue(int value) | CameraNode | O(1) | Deserializes an integer into a CameraNode instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary integration pattern involves deserializing an integer from a data source into a CameraNode object, which is then used by systems like the renderer or gameplay logic.

```java
// Deserializing a camera node identifier from a network packet
int nodeId = packet.readVarInt();
try {
    CameraNode attachmentPoint = CameraNode.fromValue(nodeId);
    
    // Pass the type-safe enum to the rendering system
    renderingSystem.setCameraAttachment(playerEntity, attachmentPoint);
} catch (ProtocolException e) {
    // Handle malformed data, e.g., disconnect the client
    log.error("Received invalid CameraNode ID: " + nodeId, e);
}
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals:** Do not rely on the `ordinal()` method for serialization. The integer values are explicitly defined and decoupled from declaration order. Using `ordinal()` would create a brittle system that breaks if the enum order is changed.
- **Ignoring Exceptions:** Failure to catch the ProtocolException thrown by fromValue will result in an unhandled exception that can crash a network thread or game loop when processing invalid data. Always wrap calls to fromValue in a try-catch block when the input source is not trusted.

## Data Pipeline
CameraNode serves as a deserialization and validation step in the data pipeline, converting raw data into a validated, type-safe object for engine consumption.

> Flow:
> Network Packet / Save File -> Raw Integer ID -> **CameraNode.fromValue(id)** -> Validated CameraNode Instance -> Rendering System / Game Logic

