---
description: Architectural reference for BuilderToolAction
---

# BuilderToolAction

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Utility

## Definition
```java
// Signature
public enum BuilderToolAction {
```

## Architecture & Concepts
BuilderToolAction is a protocol-level enumeration that provides a type-safe contract for in-game builder tool operations between the client and server. It translates abstract actions, such as "undo" or "copy selection", into fixed integer values suitable for network serialization.

This enum's primary architectural role is to eliminate the use of "magic numbers" within the networking and game logic layers. By encapsulating the raw integer protocol value, it ensures that any code handling builder tool commands is readable, maintainable, and less prone to errors. The static factory method, fromValue, serves as the primary deserialization point, converting incoming integer data from a network packet back into a strongly-typed object.

This class is fundamental to the command pattern used by the builder tools, where each enum constant represents a distinct, executable command.

### Lifecycle & Ownership
- **Creation:** All enum constants are instantiated by the Java Virtual Machine during class loading. They are compile-time constants and exist before any game code is executed.
- **Scope:** As static final instances, they persist for the entire lifetime of the application. They are effectively global, immutable singletons.
- **Destruction:** The enum and its constants are unloaded from memory only when the JVM shuts down. There is no manual lifecycle management.

## Internal State & Concurrency
- **State:** BuilderToolAction is immutable. The internal integer *value* is a final field assigned at creation and cannot be modified. The static VALUES array is also effectively immutable.
- **Thread Safety:** This enum is inherently thread-safe. Its immutable nature guarantees that it can be safely accessed and shared across any thread without synchronization. The fromValue method is a pure function and is also safe for concurrent use.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value for network serialization. |
| fromValue(int value) | BuilderToolAction | O(1) | Deserializes an integer from a network packet into a BuilderToolAction. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is used during the serialization and deserialization of packets that transmit builder tool commands.

**Serialization (Sending a command):**
```java
// When building a packet to send to the server
BuilderToolPacket packet = new BuilderToolPacket();
packet.setAction(BuilderToolAction.SelectionCopy); // The serializer will later call getValue()
```

**Deserialization (Receiving a command):**
```java
// When a packet handler processes an incoming packet
int rawAction = buffer.readVarInt();
BuilderToolAction action = BuilderToolAction.fromValue(rawAction);
handleBuilderAction(action);
```

### Anti-Patterns (Do NOT do this)
- **Using ordinal():** Do not use the built-in `ordinal()` method for serialization. The integer `value` is the canonical network representation. The order of enum constants may change during development, which would break network compatibility if `ordinal()` were used.
- **Manual Value Comparison:** Avoid comparing actions by their raw integer values. The enum provides type-safe equality checks.
    - **BAD:** `if (action.getValue() == 3)`
    - **GOOD:** `if (action == BuilderToolAction.HistoryUndo)`

## Data Pipeline
BuilderToolAction acts as a translation layer at the boundary of the network protocol and the game logic.

> **Outbound Flow (Serialization):**
> Game Logic Command -> BuilderToolPacket(`BuilderToolAction.HistoryUndo`) -> Protocol Serializer calls `getValue()` -> **int 3** -> Network Buffer

> **Inbound Flow (Deserialization):**
> Network Buffer -> Protocol Deserializer reads **int 3** -> `BuilderToolAction.fromValue(3)` -> `BuilderToolAction.HistoryUndo` -> Game Logic Handler

