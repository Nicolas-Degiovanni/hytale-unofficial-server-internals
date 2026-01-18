---
description: Architectural reference for CombatTextEntityUIAnimationEventType
---

# CombatTextEntityUIAnimationEventType

**Package:** com.hypixel.hytale.protocol
**Type:** Enumeration / Value Type

## Definition
```java
// Signature
public enum CombatTextEntityUIAnimationEventType {
```

## Architecture & Concepts
CombatTextEntityUIAnimationEventType is a type-safe enumeration that defines the distinct types of animations applicable to combat text UI elements. Its primary role is to act as a data contract within the network protocol, ensuring that both the client and server have a shared, unambiguous understanding of animation instructions.

This enum translates low-level integer identifiers, which are efficient for network transmission, into high-level, self-documenting constants used by the game's animation and rendering systems. By centralizing these definitions, it eliminates the use of "magic numbers" in the codebase, thereby increasing maintainability and reducing the risk of desynchronization errors between the client and server. It is a fundamental building block for deserializing packets related to in-game visual feedback.

## Lifecycle & Ownership
- **Creation:** The enum constants (Scale, Position, Opacity) are instantiated automatically by the Java Virtual Machine (JVM) during the class loading phase. This process is guaranteed to occur exactly once before any code attempts to access them.
- **Scope:** Application-wide. The instances are static, final, and globally accessible throughout the entire application lifetime.
- **Destruction:** The enum and its constants are unloaded and garbage collected by the JVM during application shutdown. There is no manual memory management or destruction required.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant holds a final integer value that is assigned at creation and can never be changed. The state of the enum as a whole is fixed at compile time.
- **Thread Safety:** **Inherently thread-safe**. Due to their immutable and globally unique nature, enum constants can be safely read and passed between any number of threads without requiring locks or other synchronization primitives. The static factory method fromValue is also thread-safe.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization between the enum type and its underlying integer representation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Serializes the enum constant to its network-protocol integer value. |
| fromValue(int value) | CombatTextEntityUIAnimationEventType | O(1) | Deserializes an integer from a network packet into a specific enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is almost exclusively used during the deserialization of network packets to drive client-side animation logic. The fromValue method is the designated entry point.

```java
// In a packet handler, after reading an integer from the network buffer
int rawAnimationType = networkBuffer.readVarInt();
try {
    CombatTextEntityUIAnimationEventType animationType = CombatTextEntityUIAnimationEventType.fromValue(rawAnimationType);

    // Pass the strongly-typed enum to the animation system
    uiAnimationService.triggerAnimation(entityId, animationType);
} catch (ProtocolException e) {
    // Handle corrupted or mismatched protocol data
    log.error("Received invalid CombatTextEntityUIAnimationEventType: " + rawAnimationType);
}
```

### Anti-Patterns (Do NOT do this)
- **Comparison by Integer Value:** Never use `event.getValue() == 0` for logic checks. This defeats the purpose of type safety and makes the code brittle. Always compare the enum instances directly: `event == CombatTextEntityUIAnimationEventType.Scale`.
- **Ignoring ProtocolException:** The ProtocolException thrown by fromValue is a critical signal of data corruption or a client/server version mismatch. Swallowing this exception can lead to silent failures or unpredictable rendering behavior. It must be caught and handled appropriately, often by disconnecting the client.

## Data Pipeline
This enum serves as a deserialization and translation point in the flow of data from the network to the rendering engine.

> Flow:
> Network Packet (contains integer) -> Protocol Decoder -> **CombatTextEntityUIAnimationEventType.fromValue()** -> Animation System -> UI Rendering Logic

