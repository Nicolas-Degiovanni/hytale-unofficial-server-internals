---
description: Architectural reference for UVMotionCurveType
---

# UVMotionCurveType

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum UVMotionCurveType {
```

## Architecture & Concepts
UVMotionCurveType is a static enumeration that defines a fixed set of animation easing functions. It serves as a critical component of the Hytale **protocol layer**, providing a type-safe and compact representation for motion curves used in rendering, likely for texture UV animations or particle effects.

The primary architectural function of this enum is to act as a data contract between the client and server. Instead of transmitting verbose string identifiers, the system serializes these concepts into a single integer via the `getValue` method. The `fromValue` static factory method provides the corresponding deserialization mechanism. This design ensures minimal network overhead while maintaining code readability and type safety on both endpoints.

Any modification to this enum, such as adding, removing, or reordering constants, constitutes a breaking change to the network protocol and requires a synchronized update to both client and server applications.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated once by the Java Virtual Machine during class loading. They are compile-time constants and are not created dynamically.
- **Scope:** Application-wide. These constants exist for the entire lifetime of the application process.
- **Destruction:** The constants are garbage collected when the class loader is unloaded, which typically occurs only upon application shutdown.

## Internal State & Concurrency
- **State:** UVMotionCurveType is **immutable**. Each enum constant holds a final integer value that is assigned at creation and can never be changed.
- **Thread Safety:** This class is inherently **thread-safe**. As immutable, static constants, instances of UVMotionCurveType can be safely accessed and shared across any number of threads without requiring synchronization primitives. The static factory method `fromValue` is also thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value corresponding to the enum constant, used for serialization. |
| fromValue(int value) | UVMotionCurveType | O(1) | Deserializes an integer into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The most common use case involves deserializing an integer received from the network into a type-safe enum constant. This pattern ensures that incoming data is valid before it is passed to other game systems like the rendering engine.

```java
// How a developer should normally use this
int curveTypeFromNetwork = readIntFromPacket();
try {
    UVMotionCurveType motionCurve = UVMotionCurveType.fromValue(curveTypeFromNetwork);
    // Pass motionCurve to the rendering or animation system
    renderer.setAnimationCurve(motionCurve);
} catch (ProtocolException e) {
    // Handle invalid data, e.g., disconnect the client
    log.error("Received invalid UVMotionCurveType: " + curveTypeFromNetwork);
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Integer Comparison:** Do not use magic numbers in your code. Rely on the enum constants for clarity and safety. The underlying integer values are an implementation detail of the protocol and should not leak into game logic.
  ```java
  // BAD: Prone to error if enum order changes
  if (particle.curveType == 1) { ... }

  // GOOD: Type-safe and readable
  if (particle.curveType == UVMotionCurveType.IncreaseLinear) { ... }
  ```
- **Bypassing Deserialization:** Do not manually cast or map integers to the enum without using the provided `fromValue` factory. Doing so bypasses critical validation logic and can lead to an ArrayIndexOutOfBoundsException if the integer is invalid.

## Data Pipeline
UVMotionCurveType is not an active component in a pipeline; rather, it is the data itself that flows through the system. It represents a piece of serialized state.

> **Serialization Flow (Server -> Client):**
> Game State -> Particle Effect with **UVMotionCurveType.IncreaseLinear** -> Protocol Serializer (`getValue()`) -> Network Packet (contains integer `1`) -> Client

> **Deserialization Flow (Client):**
> Network Packet (contains integer `1`) -> Protocol Deserializer (`fromValue(1)`) -> **UVMotionCurveType.IncreaseLinear** -> Rendering Engine -> Visual Effect on Screen

