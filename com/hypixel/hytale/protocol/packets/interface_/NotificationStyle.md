---
description: Architectural reference for NotificationStyle
---

# NotificationStyle

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Utility

## Definition
```java
// Signature
public enum NotificationStyle {
```

## Architecture & Concepts
The NotificationStyle enum serves as a strict, type-safe contract for representing UI notification severities within the network protocol. It translates a simple integer, which is efficient for network transfer, into a self-documenting and robust type on both the client and server.

This component is fundamental to the packet deserialization pipeline. Its primary role is to prevent "magic numbers" (e.g., `if (notificationType == 2)`) in higher-level game logic. By enforcing a valid range of values through the static factory method fromValue, it guarantees that any consuming system, such as the UI renderer, will only ever receive a valid, known style. An invalid integer from a packet will result in a ProtocolException, immediately halting the processing of the malformed data at the boundary layer, thus preventing state corruption.

## Lifecycle & Ownership
- **Creation:** All enum constants (Default, Danger, etc.) are instantiated by the Java Virtual Machine during class loading. They are compile-time constants, not runtime objects in the traditional sense.
- **Scope:** As static final instances, they exist for the entire lifetime of the application once the NotificationStyle class is loaded.
- **Destruction:** The enum instances are reclaimed by the JVM only when the application's class loader is garbage collected, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** NotificationStyle is deeply immutable. Its internal integer value is final, and the enum constants themselves are singletons. The static VALUES array is a cache for the `values()` result, providing a minor performance optimization for the fromValue method.
- **Thread Safety:** This enum is inherently thread-safe. Its immutability guarantees that it can be safely read, passed, and compared across any number of threads without requiring synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Serializes the enum constant to its network-protocol integer representation. |
| fromValue(int value) | NotificationStyle | O(1) | Deserializes an integer from the network into a NotificationStyle instance. Throws ProtocolException if the value is out of the defined range. |

## Integration Patterns

### Standard Usage
The primary integration pattern involves deserializing an integer from a network packet into a NotificationStyle object. This object is then passed to UI systems for rendering.

```java
// Example: Processing a notification packet
int styleId = packet.readVarInt(); // Read integer from network stream
try {
    NotificationStyle style = NotificationStyle.fromValue(styleId);
    uiManager.displayNotification("Welcome!", style);
} catch (ProtocolException e) {
    // Handle malformed packet, e.g., disconnect the client
    log.error("Received invalid notification style: " + styleId);
}
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals:** Do not use the built-in `ordinal()` method for serialization. The integer `value` is the explicit network contract. The ordinal is an implementation detail that can change if the enum order is modified, leading to critical deserialization bugs.
- **Manual Comparison:** Avoid comparing raw integer values in game logic. The purpose of the enum is to provide type safety.

```java
// BAD: Brittle and ignores the type system
if (style.getValue() == 2) {
    // ... logic for Warning
}

// GOOD: Robust and self-documenting
if (style == NotificationStyle.Warning) {
    // ... logic for Warning
}
```

## Data Pipeline
NotificationStyle acts as a deserialization and validation gate for data flowing from the network to the user interface.

> Flow:
> Network Byte Stream -> Packet Deserializer -> `NotificationStyle.fromValue(intValue)` -> **NotificationStyle Enum** -> UI Rendering System

