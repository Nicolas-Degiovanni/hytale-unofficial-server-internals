---
description: Architectural reference for ShaderType
---

# ShaderType

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum ShaderType {
```

## Architecture & Concepts
The ShaderType enumeration serves as a type-safe, statically-defined contract for all supported visual shader effects within the rendering engine. Its primary architectural role is to decouple the game logic and network protocol from the low-level implementation details of the graphics pipeline.

By mapping human-readable names like *Wind* or *Water* to fixed integer identifiers, ShaderType prevents the use of "magic numbers" in the codebase. This is critical for network serialization, where a compact and stable integer representation is required for transmission between the client and server. The class acts as a translation layer, ensuring that when the server specifies shader `1`, the client reliably interprets this as the *Wind* effect.

This enumeration is a foundational component of the game's visual data model and is referenced by any system that defines or applies visual effects to game objects, such as block renderers, entity animators, and particle systems.

### Lifecycle & Ownership
- **Creation:** All instances of ShaderType (None, Wind, Water, etc.) are constructed automatically by the Java Virtual Machine (JVM) when the ShaderType class is first loaded into memory. This process is managed entirely by the JVM classloader.
- **Scope:** ShaderType instances are static and global. They exist for the entire lifetime of the application. They are, in effect, permanent singletons for each defined type.
- **Destruction:** The instances are garbage collected only when the application's classloader is unloaded, which typically happens only at application shutdown. There is no manual memory management required.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a single, final integer field named *value*. The static *VALUES* array is also final and is populated once during class initialization. The state of this class cannot be modified at runtime.
- **Thread Safety:** This class is intrinsically thread-safe. As all its instances and internal fields are immutable and initialized during class loading, it can be safely accessed and read from any number of concurrent threads without synchronization or locking mechanisms.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the stable integer identifier for the enum constant. Used for serialization. |
| fromValue(int value) | ShaderType | O(1) | **Static Factory.** Deserializes an integer into a ShaderType instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary integration pattern involves using the static factory method *fromValue* to deserialize data from a network stream or data file, and the instance method *getValue* to serialize it.

```java
// Deserialization from a network payload
int shaderId = networkBuffer.readInt();
ShaderType effect = ShaderType.fromValue(shaderId);

// Applying the shader type in the render system
renderer.applyShader(effect);

// Serialization for sending to a client
int idToSend = myEntity.getShaderType().getValue();
networkBuffer.writeInt(idToSend);
```

### Anti-Patterns (Do NOT do this)
- **Integer Comparison:** Do not use the raw integer value for logic checks. This defeats the purpose of type safety and creates brittle code that is hard to refactor.
    - **BAD:** `if (shader.getValue() == 6) { ... }`
    - **GOOD:** `if (shader == ShaderType.Water) { ... }`
- **Invalid Deserialization:** Do not bypass the *fromValue* method. Manually casting or looking up the value in the *VALUES* array circumvents the critical bounds-checking and error handling.
    - **BAD:** `ShaderType type = ShaderType.VALUES[rawValue]; // Throws ArrayIndexOutOfBoundsException`
    - **GOOD:** `ShaderType type = ShaderType.fromValue(rawValue); // Throws a clean ProtocolException`

## Data Pipeline
ShaderType is a key component in the data serialization and rendering pipeline, acting as a point of translation between the network protocol and the rendering engine.

> Flow:
> Network Packet -> Protocol Decoder -> **ShaderType.fromValue(intValue)** -> Game State Object -> Render System -> GPU Shader Program

