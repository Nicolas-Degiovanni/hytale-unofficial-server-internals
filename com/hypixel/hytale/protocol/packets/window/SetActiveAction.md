---
description: Architectural reference for SetActiveAction
---

# SetActiveAction

**Package:** com.hypixel.hytale.protocol.packets.window
**Type:** Data Transfer Object (DTO) / Transient

## Definition
```java
// Signature
public class SetActiveAction extends WindowAction {
```

## Architecture & Concepts
The SetActiveAction class is a highly specialized network packet component within the Hytale Protocol Layer. It represents a minimal, single-byte message designed to communicate a binary state change (true or false) for a user interface element, as indicated by its parent class WindowAction.

This class is not a service or a manager; it is a pure data structure. Its primary architectural role is to serve as an immutable message for state synchronization between the client and server. The design prioritizes network efficiency and serialization performance over flexibility. The fixed size of one byte makes it ideal for high-frequency, low-latency UI interactions where payload size is critical.

It is a concrete implementation of a command pattern, where the object itself encapsulates all information needed to execute an action—in this case, setting a UI component to an active or inactive state.

## Lifecycle & Ownership
- **Creation:**
    - **Outbound (Sending):** Instantiated directly by game logic when a state change must be transmitted. For example, a UI controller creates a new SetActiveAction to send to the remote peer.
    - **Inbound (Receiving):** Instantiated by the protocol's deserialization pipeline. The static factory method **deserialize** is the sole entry point for creating an instance from an incoming network ByteBuf.
- **Scope:** Extremely short-lived and transient. An instance exists only for the brief moment it is being serialized into a buffer or deserialized and passed to a handler. It is not intended to be stored or referenced long-term.
- **Destruction:** The object becomes eligible for garbage collection immediately after its data has been processed by a handler or written to the network channel. There is no manual destruction or pooling mechanism.

## Internal State & Concurrency
- **State:** The class holds a single mutable boolean field, **state**. While technically mutable, instances should be treated as immutable after creation. Modifying the state after instantiation, especially before serialization, is a significant anti-pattern.
- **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context, such as a Netty event loop or the main game thread. Accessing or modifying an instance from multiple threads without external synchronization will result in undefined behavior and data corruption.

## API Surface
The public API is focused exclusively on serialization, deserialization, and data integrity checks.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf buf) | int | O(1) | Encodes the internal boolean state into the provided network buffer. Returns the number of bytes written (always 1). |
| deserialize(ByteBuf buf, int offset) | SetActiveAction | O(1) | **Static Factory.** Constructs a new SetActiveAction instance by reading one byte from the buffer at the given offset. |
| computeSize() | int | O(1) | Returns the constant size of the packet's payload, which is always 1 byte. |
| validateStructure(ByteBuf buffer, int offset) | ValidationResult | O(1) | **Static Validator.** Performs a pre-deserialization check to ensure the buffer contains enough readable bytes to form a valid packet. |

## Integration Patterns

### Standard Usage
This object is created on-the-fly and passed to a network-aware system for transmission. It is never managed or retrieved from a central registry.

```java
// Example: A UI button click handler sending a state change to the server.
// The 'window' object here is a hypothetical UI controller with network capabilities.

boolean isNowActive = true;
window.sendAction(new SetActiveAction(isNowActive));
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not create a single SetActiveAction and reuse it for multiple transmissions. This is unsafe, especially in asynchronous environments. Always create a new instance for each distinct action.
- **Manual Deserialization:** Never attempt to read from a ByteBuf and populate a new SetActiveAction manually. The static **deserialize** method is the only supported pathway for creating an instance from network data.
- **Cross-Thread Modification:** Do not create an instance on one thread and pass it to a network thread for serialization if the original thread might modify it. The state must be considered final upon creation.

## Data Pipeline
SetActiveAction serves as a data payload within the network protocol. Its flow is linear and unidirectional within any single transaction.

> **Outbound Flow (e.g., Client to Server):**
> UI Event -> Game Logic creates `new SetActiveAction(true)` -> Protocol Encoder invokes **serialize()** -> Raw Bytes written to Netty Channel -> Network

> **Inbound Flow (e.g., Server to Client):**
> Network -> Raw Bytes read from Netty Channel -> Protocol Decoder invokes **deserialize()** -> Fully formed SetActiveAction object -> Event Bus -> UI System updates component state

