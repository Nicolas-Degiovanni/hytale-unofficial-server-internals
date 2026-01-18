---
description: Architectural reference for ClientFeature
---

# ClientFeature

**Package:** com.hypixel.hytale.protocol.packets.setup
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum ClientFeature {
   SplitVelocity(0),
   Mantling(1),
   SprintForce(2),
   CrouchSlide(3),
   SafetyRoll(4),
   DisplayHealthBars(5),
   DisplayCombatText(6);
```

## Architecture & Concepts
ClientFeature is a static, type-safe enumeration that defines a contract of gameplay capabilities between the client and the server. Its primary role is within the network protocol layer, specifically during the connection setup and handshake sequence. Each enum constant represents a distinct client-side feature, such as a movement mechanic or a UI element, that the server needs to be aware of.

The integer value associated with each feature is its wire format representation. This design ensures that the network payload is compact and efficient, while the code on both client and server remains readable and robust by avoiding the use of "magic numbers". The enum effectively serves as a serialization and deserialization dictionary for a specific set of boolean flags communicated during login.

## Lifecycle & Ownership
- **Creation:** All enum instances are created and initialized by the Java Virtual Machine (JVM) during class loading. They are compile-time constants, not runtime objects in the traditional sense.
- **Scope:** The instances are static and persist for the entire lifetime of the application. They are effectively global, immutable singletons.
- **Destruction:** The instances are garbage collected only when the JVM shuts down. There is no manual memory management required.

## Internal State & Concurrency
- **State:** Immutable. The internal integer value of each enum constant is declared final and assigned at creation. Its state cannot be modified post-initialization.
- **Thread Safety:** ClientFeature is inherently thread-safe. The JVM guarantees that enum initialization is a thread-safe process. Due to their immutability, instances can be freely shared and read across multiple threads without any need for external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the feature for network serialization. |
| fromValue(int value) | ClientFeature | O(1) | Deserializes an integer from a network packet into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used by packet serializers and deserializers to convert between the network representation and the in-memory game state.

```java
// Deserializing a feature flag from a network buffer
int featureId = buffer.readVarInt();
ClientFeature feature = ClientFeature.fromValue(featureId);

// Logic based on the feature
if (feature == ClientFeature.Mantling) {
    player.getCapabilities().setCanMantle(true);
}

// Serializing a feature for transmission
ClientFeature featureToSend = ClientFeature.SprintForce;
buffer.writeVarInt(featureToSend.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Using Integer Values in Logic:** Do not compare against the raw integer value in game logic. This creates brittle code that is hard to refactor and understand. The integer value is an implementation detail of the network protocol only.
  - **BAD:** `if (feature.getValue() == 1) { // logic }`
  - **GOOD:** `if (feature == ClientFeature.Mantling) { // logic }`
- **Ignoring ProtocolException:** The fromValue method can throw a ProtocolException. This must be caught at the network boundary to handle data corruption or connections from incompatible clients. Failure to do so can crash the session handler.

## Data Pipeline
ClientFeature acts as a translation layer between the raw network stream and the server's internal representation of client capabilities.

**Inbound (Deserialization):**
> Network Byte Stream -> Protocol Decoder -> `int value` -> **ClientFeature.fromValue(value)** -> Server-Side Player Capability State

**Outbound (Serialization):**
> Client-Side Capability State -> `ClientFeature.Mantling` -> **feature.getValue()** -> `int value` -> Protocol Encoder -> Network Byte Stream

