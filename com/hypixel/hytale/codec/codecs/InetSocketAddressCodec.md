---
description: Architectural reference for InetSocketAddressCodec
---

# InetSocketAddressCodec

**Package:** com.hypixel.hytale.codec.codecs
**Type:** Transient

## Definition
```java
// Signature
public class InetSocketAddressCodec implements Codec<InetSocketAddress> {
```

## Architecture & Concepts
The InetSocketAddressCodec is a specialized component within the engine's serialization framework. Its sole responsibility is to translate the low-level Java networking object, InetSocketAddress, into a standardized, human-readable string representation for data interchange formats like BSON and JSON, and vice-versa.

This class acts as a critical bridge between network configuration data and the core Java networking stack. By abstracting the serialization logic, it ensures that network addresses are consistently formatted as *host:port* or just *port* across all parts of the engine, from server configuration files to network packets.

A key architectural feature is its configurable nature. Each instance is constructed with a *defaultPort*, which provides resilience when parsing address strings where the port is omitted. This allows different parts of the system (e.g., a game client vs. a master server) to use different default ports while sharing the same core parsing logic.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly in application code. They are instantiated and configured by a central CodecRegistry or a dependency injection container during the application's bootstrap phase. The `defaultPort` is supplied from a configuration source at this time.
- **Scope:** The lifetime of an InetSocketAddressCodec instance is bound to the CodecRegistry that manages it. It is designed to be stateless and reusable, typically persisting for the entire application session.
- **Destruction:** The object is eligible for garbage collection when its parent CodecRegistry is destroyed, which normally occurs during a controlled application shutdown.

## Internal State & Concurrency
- **State:** This class is effectively immutable. Its only instance field, `defaultPort`, is final and is set exclusively at construction time. It performs no internal caching or state modification during its operations.
- **Thread Safety:** The InetSocketAddressCodec is inherently thread-safe. All of its methods are pure functions of their inputs and the immutable `defaultPort` field. It can be safely shared and invoked concurrently by multiple threads without any external locking or synchronization mechanisms.

## API Surface
The public API is designed for integration with the parent Codec framework, not for direct invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | InetSocketAddress | O(1) | Deserializes a BSON string into an InetSocketAddress object. |
| encode(address, extraInfo) | BsonValue | O(1) | Serializes an InetSocketAddress object into a BSON string. |
| decodeJson(reader, extraInfo) | InetSocketAddress | O(1) | Deserializes a JSON string from a stream into an InetSocketAddress. |
| toSchema(context) | Schema | O(1) | Generates a StringSchema definition with a regex pattern for validating network addresses. |

## Integration Patterns

### Standard Usage
This codec is designed to be used implicitly by the serialization system. A developer will never need to interact with it directly. The framework automatically selects this codec when it encounters an InetSocketAddress field during a serialization or deserialization operation.

```java
// CORRECT: The framework finds and uses the registered codec automatically.
// Assume 'codecService' is the central serialization authority.

// Serialization
ServerInfo info = new ServerInfo();
info.setAddress(new InetSocketAddress("play.hytale.com", 25565));
BsonValue bson = codecService.encode(info); // The codec is used here

// Deserialization
ServerInfo decodedInfo = codecService.decode(bson, ServerInfo.class); // And here
InetSocketAddress address = decodedInfo.getAddress();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually create instances of this class within business logic. The codec system is responsible for its lifecycle.
  ```java
  // INCORRECT
  InetSocketAddressCodec myCodec = new InetSocketAddressCodec(25565);
  InetSocketAddress addr = myCodec.decode(...);
  ```
- **Manual String Parsing:** Do not replicate the logic of this class. Rely on the central codec system to ensure consistent address handling.

## Data Pipeline
The InetSocketAddressCodec is a bidirectional data transformation component.

**Deserialization Flow (Reading Data)**
> BSON String ("127.0.0.1:25565") -> **InetSocketAddressCodec.decode** -> Java InetSocketAddress Object

**Serialization Flow (Writing Data)**
> Java InetSocketAddress Object -> **InetSocketAddressCodec.encode** -> BSON String ("127.0.0.1:25565")

