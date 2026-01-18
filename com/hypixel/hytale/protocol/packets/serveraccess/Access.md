---
description: Architectural reference for the Access enum, defining server visibility levels.
---

# Access

**Package:** com.hypixel.hytale.protocol.packets.serveraccess
**Type:** Utility

## Definition
```java
// Signature
public enum Access {
```

## Architecture & Concepts
The Access enum is a fundamental data contract within the Hytale network protocol. It serves as a type-safe representation of a game server's visibility and join permissions. This component is critical for establishing the rules of engagement between a client attempting to connect and a server advertising its status.

Architecturally, Access acts as a static data dictionary, translating between the human-readable concepts of server privacy (Private, LAN, Friend, Open) and their low-level integer representations used for network serialization. By encapsulating this logic, it prevents the proliferation of "magic numbers" throughout the networking and server management code, thereby increasing readability and reducing the risk of protocol-level bugs. It is a foundational building block for server discovery, matchmaking, and direct-join systems.

## Lifecycle & Ownership
As a Java enum, Access has a specialized lifecycle managed directly by the Java Virtual Machine, not by any application-level framework or dependency injection container.

- **Creation:** All enum constants (Private, LAN, Friend, Open) are instantiated automatically by the JVM when the Access class is first loaded. They are compile-time constants and exist as singleton instances.
- **Scope:** The enum constants are static and persist for the entire lifetime of the application. Their scope is global.
- **Destruction:** The instances are reclaimed only when the application's ClassLoader is garbage collected, which typically occurs at application shutdown.

**WARNING:** The lifecycle of this enum is not controlled by developers. It is guaranteed to be available once the class is loaded.

## Internal State & Concurrency
- **State:** The Access enum is **immutable**. Each constant holds a final primitive integer, and the static VALUES array is populated once during class initialization. No state can be modified at runtime.
- **Thread Safety:** This class is **inherently thread-safe**. Its immutability and the JVM's guarantees for enum initialization ensure that it can be safely accessed and used from any thread without external synchronization or locks. This is critical for high-performance, multi-threaded networking code that may deserialize packets concurrently.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Serializes the enum constant to its integer representation for network transport. |
| fromValue(int value) | Access | O(1) | Deserializes an integer from a network packet into the corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary use case is during packet serialization and deserialization. Code processing network data will use the static factory method to convert an incoming integer into a safe Access type.

```java
// Deserializing an incoming server status packet
int accessValue = packet.readVarInt();
Access serverAccess = Access.fromValue(accessValue);

// Logic based on the server's access level
if (serverAccess == Access.Open) {
    // Enable the "Join" button in the UI
}
```

### Anti-Patterns (Do NOT do this)
- **Comparison by Integer:** Never compare an Access instance by its underlying integer value. This defeats the purpose of type safety and makes the code brittle.
  - **BAD:** `if (serverAccess.getValue() == 3)`
  - **GOOD:** `if (serverAccess == Access.Open)`

- **Uncaught Exceptions:** The fromValue method is a critical validation point for network data. Failure to handle the ProtocolException it throws can lead to unhandled exceptions and potential client or server instability when processing malformed or malicious packets.
  - **BAD:** `Access serverAccess = Access.fromValue(untrustedValue);`
  - **GOOD:**
    ```java
    try {
        Access serverAccess = Access.fromValue(untrustedValue);
        // ... process valid access level
    } catch (ProtocolException e) {
        // ... handle invalid packet data, log, or disconnect
    }
    ```

## Data Pipeline
Access is not a processing stage but rather a data model that exists *within* the pipeline. It represents the transformation of a raw integer into a validated, type-safe object.

> **Inbound Flow (Deserialization):**
> Network Byte Stream -> Packet Decoder -> `int` -> **Access.fromValue()** -> Game Logic

> **Outbound Flow (Serialization):**
> Game Logic -> `Access instance` -> **access.getValue()** -> `int` -> Packet Encoder -> Network Byte Stream

