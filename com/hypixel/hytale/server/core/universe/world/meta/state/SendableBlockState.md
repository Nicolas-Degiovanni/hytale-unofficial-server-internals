---
description: Architectural reference for SendableBlockState
---

# SendableBlockState

**Package:** com.hypixel.hytale.server.core.universe.world.meta.state
**Type:** Contract / Interface

## Definition
```java
// Signature
@Deprecated
public interface SendableBlockState {
```

## Architecture & Concepts

The SendableBlockState interface defines a contract for server-side block metadata that must be synchronized with clients. It represents a behavioral pattern for any block state that requires custom network replication logic beyond simple block type and orientation. This component is a critical part of the world state replication pipeline, acting as the serialization bridge between the server's in-memory representation of a block's state and the network packets sent to players.

Implementing this interface allows a block's state object to control precisely how it is represented on the network. This is typically used for complex blocks like signs, chests, or machinery whose state cannot be fully described by a simple block ID.

**WARNING:** This interface is marked as **Deprecated**. It likely represents an older, object-oriented approach to state synchronization. Modern architectures often favor data-driven or component-based systems (like an Entity Component System) for state replication. New development should avoid implementing this interface and instead use the designated modern synchronization mechanism.

## Lifecycle & Ownership

As an interface, SendableBlockState itself has no lifecycle. The following describes the expected lifecycle of an *object that implements* this interface.

-   **Creation:** An implementing object is instantiated by the world engine when a corresponding complex block is placed or loaded from storage. For example, a `SignBlockState` object would be created when a sign is placed in the world.
-   **Scope:** The object's lifetime is tied directly to the block it represents. It persists as long as the block exists in a loaded chunk on the server.
-   **Destruction:** The object is marked for garbage collection when the block is destroyed by a player or the chunk containing it is unloaded from server memory. The `unloadFrom` method is a key part of its pre-destruction cleanup, ensuring clients are notified to remove their local representation.

## Internal State & Concurrency

The interface itself is stateless. Any implementation, however, is expected to be stateful.

-   **State:** Implementations of SendableBlockState are inherently mutable. They hold the live, server-side state of a specific block in the world (e.g., the text on a sign, the inventory of a chest).
-   **Thread Safety:** Implementations are **not expected to be thread-safe**. All interactions with a SendableBlockState object must be performed on the main server thread for the dimension in which the block resides. Access from other threads will lead to race conditions and world corruption. The world engine guarantees serialized access during its update tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| sendTo(List<Packet> var1) | void | O(N) | Serializes the block's current state into one or more network packets and adds them to the provided list. N is the complexity of the state. |
| unloadFrom(List<Packet> var1) | void | O(1) | Creates the necessary packets to instruct a client to destroy its local representation of this block's state. |
| canPlayerSee(PlayerRef player) | boolean | O(1) | A default visibility check. Allows for state to be hidden from certain players, though the default implementation always returns true. |

## Integration Patterns

### Standard Usage

This interface is not meant to be used directly by service-level code. It is implemented by specific block state classes, which are then managed by the server's chunk and world systems. The world replication manager identifies objects implementing this contract and invokes `sendTo` or `unloadFrom` as players move in and out of visibility range.

```java
// Example of a class that would implement the interface
// Do NOT call these methods directly. The engine does this.

public class SignBlockState implements SendableBlockState {
    private String[] lines;

    @Override
    public void sendTo(List<Packet> packetList) {
        // Creates a P_BLOCK_ENTITY_DATA packet with sign text
        // and adds it to the packetList.
    }

    @Override
    public void unloadFrom(List<Packet> packetList) {
        // This is often a no-op, as destroying the block
        // implicitly unloads its state on the client.
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Manual Invocation:** Never call `sendTo` or `unloadFrom` directly. These methods are exclusively for the server's world replication engine. Manually calling them can cause client de-synchronization and network instability.
-   **Stateful `canPlayerSee`:** Avoid complex or stateful logic within the `canPlayerSee` method. This check may be called frequently and should be a fast, cheap operation to prevent performance degradation on the server tick.
-   **Implementation on Non-Blocks:** Do not implement this interface on objects that do not represent the state of a single, specific block in the world. This contract is tightly coupled to the block/chunk lifecycle.

## Data Pipeline

The data flow for an implementing object is unidirectional, from the server's game state to the network layer.

> Flow:
> Server World Tick -> Block State Change (e.g., Sign text updated) -> **SendableBlockState (Implementation)** -> `sendTo` called by Replication Engine -> List of Packets -> Network Layer -> Client


