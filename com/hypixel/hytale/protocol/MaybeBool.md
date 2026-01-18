---
description: Architectural reference for MaybeBool
---

# MaybeBool

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum MaybeBool {
```

## Architecture & Concepts
MaybeBool is a tri-state boolean representation used exclusively within the Hytale network protocol layer. It serves as a type-safe, high-performance alternative to a nullable Boolean wrapper, providing three distinct states: *True*, *False*, and *Null*.

The primary design goal of this enum is to create an explicit and compact wire format for boolean values that may be optional or have an undefined state. Each state maps directly to a non-negotiable integer value (0 for Null, 1 for False, 2 for True), which is the canonical representation used during data serialization and deserialization. This avoids ambiguity and reduces payload size compared to more complex representations.

This component is fundamental to protocol stability. Any change to the underlying integer values constitutes a breaking change to the network protocol.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated once by the JVM during the initial class loading phase. The internal VALUES array is also populated at this time. These instances are effectively static singletons.
- **Scope:** Application-wide. A single set of MaybeBool constants exists for the entire runtime.
- **Destruction:** The enum instances persist for the lifetime of the application and are garbage collected only upon JVM shutdown.

## Internal State & Concurrency
- **State:** Inherently immutable. The integer value associated with each enum constant is final. The static VALUES array is a private, final cache that is populated once and never modified.
- **Thread Safety:** This class is unconditionally thread-safe. As an enum with final fields, its state cannot be mutated after initialization. It can be safely accessed and used by any thread without external synchronization.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value for serialization. This is the canonical wire format representation. |
| fromValue(int value) | MaybeBool | O(1) | Deserializes an integer from a network stream into a MaybeBool instance. Throws ProtocolException if the value is out of the defined range [0-2]. |

## Integration Patterns

### Standard Usage
This enum should be used when reading from or writing to a network buffer where a tri-state boolean is expected.

```java
// Deserializing a value from a network buffer
int valueFromStream = buffer.readVarInt();
MaybeBool state = MaybeBool.fromValue(valueFromStream);

// Serializing a value to a network buffer
MaybeBool currentState = MaybeBool.True;
buffer.writeVarInt(currentState.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Using ordinal() for Serialization:** **CRITICAL WARNING:** Never use the built-in `ordinal()` method for serialization. The protocol contract is bound to the explicit integer `value` field. The order of enum declarations could change in the future, which would alter the ordinal but not the value, silently breaking network compatibility.
- **Null Checks:** Do not check for `myMaybeBool == null`. The undefined state is explicitly represented by the `MaybeBool.Null` constant. A variable of type MaybeBool should never be null in a correctly functioning system.

## Data Pipeline
MaybeBool acts as a low-level data-mapping component between the raw network stream and higher-level game logic.

> **Inbound Flow (Deserialization):**
> Network Byte Stream -> Protocol Decoder (reads integer) -> `MaybeBool.fromValue(intValue)` -> **MaybeBool** instance -> Game Logic

> **Outbound Flow (Serialization):**
> Game Logic -> **MaybeBool** instance -> `maybeBool.getValue()` -> Integer Value -> Protocol Encoder (writes integer) -> Network Byte Stream

