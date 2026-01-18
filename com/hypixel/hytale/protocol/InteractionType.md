---
description: Architectural reference for InteractionType
---

# InteractionType

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum InteractionType {
```

## Architecture & Concepts
The InteractionType enumeration is a foundational component of the Hytale network protocol. It serves as a strict data contract, translating high-level gameplay actions and events into a compact, serializable integer format suitable for network transmission.

This class establishes a canonical "vocabulary" for all entity and world interactions. By defining a fixed set of possible actions—such as Primary, CollisionEnter, or Death—it eliminates ambiguity between the client and server. When a player performs an action, the client serializes the corresponding InteractionType enum constant into its integer value. The server receives this integer and deserializes it back into the same InteractionType constant, ensuring both ends of the connection operate on an identical and validated understanding of the event.

This design is critical for protocol stability, network efficiency, and forward compatibility. It prevents the use of "magic numbers" in the game logic and provides a single, authoritative source for all interaction definitions.

### Lifecycle & Ownership
- **Creation:** All InteractionType constants are instantiated by the Java Virtual Machine during class loading. They are compile-time constants and are not created dynamically during runtime.
- **Scope:** As static final instances, they exist for the entire lifetime of the application. They are globally accessible and immutable.
- **Destruction:** The constants are garbage collected only when the application's ClassLoader is unloaded, which typically occurs at JVM shutdown.

## Internal State & Concurrency
- **State:** InteractionType is immutable. Each enum constant holds a final integer value that is assigned at compile time. The static VALUES array is a performance optimization to cache the result of the expensive values() method call, and it is also final.
- **Thread Safety:** This class is intrinsically thread-safe. Its immutable nature guarantees that its state cannot be corrupted by concurrent access from multiple threads. The static fromValue method is a pure function and can be called concurrently without issue.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromValue(int value) | static InteractionType | O(1) | Deserializes an integer from a network stream into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |
| getValue() | int | O(1) | Serializes the enum constant into its integer representation for network transmission. |

## Integration Patterns

### Standard Usage
InteractionType is primarily used for deserializing incoming network packets and routing logic via switch statements.

```java
// How to deserialize and process an interaction from a network buffer
int interactionId = buffer.readVarInt();
InteractionType type = InteractionType.fromValue(interactionId);

switch (type) {
    case Primary:
        // Handle primary attack logic
        break;
    case CollisionEnter:
        // Handle entity collision logic
        break;
    // ... other cases
}
```

### Anti-Patterns (Do NOT do this)
- **Using Magic Numbers:** Never compare against the raw integer value. The purpose of the enum is to provide type safety and readability.
    - **BAD:** `if (interactionId == 8) { ... }`
    - **GOOD:** `if (type == InteractionType.CollisionEnter) { ... }`
- **Manual Deserialization:** Do not attempt to validate the integer value manually. The static fromValue method centralizes the protocol's validation logic.
    - **BAD:** `InteractionType type = InteractionType.VALUES[interactionId]; // Unsafe, can throw ArrayIndexOutOfBoundsException`
    - **GOOD:** `InteractionType type = InteractionType.fromValue(interactionId); // Safe, throws a specific ProtocolException`

## Data Pipeline
InteractionType is a critical link in the data pipeline for game events, acting as the translation layer between the game logic and the raw network stream.

> **Outbound Flow (Serialization):**
> Game Event -> **InteractionType.Primary** -> `getValue()` -> `0` -> Network Packet

> **Inbound Flow (Deserialization):**
> Network Packet -> `0` -> `fromValue(0)` -> **InteractionType.Primary** -> Game Logic (Switch)

