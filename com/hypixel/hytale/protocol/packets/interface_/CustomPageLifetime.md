---
description: Architectural reference for CustomPageLifetime
---

# CustomPageLifetime

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Utility

## Definition
```java
// Signature
public enum CustomPageLifetime {
```

## Architecture & Concepts
The CustomPageLifetime enum is a type-safe enumeration that defines the dismissal behavior of custom user interface pages sent via the network protocol. It serves as a critical component in the contract between the server and client for UI interactions.

Its primary architectural function is to replace ambiguous "magic numbers" with strongly-typed, self-documenting constants. This prevents a class of bugs related to incorrect integer values being sent over the network to represent UI behavior. The enum acts as both a serialization helper (providing an integer value) and a deserialization and validation gate (parsing an integer back into a safe type).

By centralizing the valid lifetime policies, this enum ensures that any code creating or parsing UI-related packets operates on a known, valid set of behaviors. The inclusion of a `ProtocolException` during deserialization from an invalid value makes it a key part of the protocol's data integrity and security model, immediately rejecting malformed or malicious packets.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine during class loading. They are compile-time constants and are not created dynamically.
- **Scope:** Application-wide. As static final instances, these constants persist for the entire lifetime of the application.
- **Destruction:** The constants are destroyed only when the application's class loader is garbage collected, which typically occurs at shutdown.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a private final integer value that is set at creation and can never be changed. The static `VALUES` array is also effectively immutable.
- **Thread Safety:** This enum is inherently thread-safe. Its immutable nature guarantees that it can be safely accessed and shared across any number of threads without requiring synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value for network serialization. |
| fromValue(int value) | static CustomPageLifetime | O(1) | Deserializes an integer into an enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is used when constructing or deconstructing packets that define custom UI pages. The server uses `getValue` to serialize the policy, and the client uses `fromValue` to deserialize and apply it.

```java
// Server-side: Serializing a UI policy for a packet
int lifetimePolicy = CustomPageLifetime.CanDismiss.getValue();
packet.writeInt(lifetimePolicy);

// Client-side: Deserializing the policy from a packet
int receivedValue = packet.readInt();
CustomPageLifetime lifetime = CustomPageLifetime.fromValue(receivedValue);
uiManager.applyLifetimePolicy(lifetime);
```

### Anti-Patterns (Do NOT do this)
- **Using Magic Numbers:** Do not use raw integers in game logic to represent lifetime policies. This defeats the purpose of the enum and introduces fragility.
- **Ignoring Exceptions:** Failure to catch the `ProtocolException` thrown by `fromValue` can lead to an unhandled exception in the network processing thread, potentially disconnecting the client. Always wrap calls to `fromValue` in a try-catch block during packet parsing.

## Data Pipeline
CustomPageLifetime acts as a translation and validation point for data moving between high-level game logic and the low-level network buffer.

> **Serialization Flow:**
> UI Builder -> **CustomPageLifetime.CanDismiss** -> `getValue()` -> `1` (int) -> Network Buffer

> **Deserialization Flow:**
> Network Buffer -> `1` (int) -> `fromValue(1)` -> **CustomPageLifetime.CanDismiss** -> UI System

