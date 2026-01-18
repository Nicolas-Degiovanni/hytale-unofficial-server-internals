---
description: Architectural reference for MovementEffects
---

# MovementEffects

**Package:** com.hypixel.hytale.server.core.asset.modifiers
**Type:** Transient Data Object

## Definition
```java
// Signature
public class MovementEffects implements NetworkSerializable<com.hypixel.hytale.protocol.MovementEffects> {
```

## Architecture & Concepts
MovementEffects is a server-side data container that represents a set of movement restrictions which can be applied to an entity. It is not a service or manager, but rather a Plain Old Java Object (POJO) that encapsulates state related to player input control.

Its primary architectural role is to act as a bridge between declarative game data (defined in asset files) and the server's runtime logic. This is achieved through its static **CODEC** field, a powerful `BuilderCodec` that defines how an instance is serialized and deserialized. This allows game designers to define complex movement-impairing effects in configuration files, which are then loaded into the game engine as fully-formed MovementEffects objects.

The implementation of the NetworkSerializable interface signifies its role in the server-to-client data synchronization pipeline. The object can transform its server-side state into a network-ready packet to enforce these movement restrictions on the client.

The `appendInherited` pattern used within the CODEC suggests that these effects are designed to be part of a hierarchical or cascading system, where a specific effect can inherit and override properties from a more general one.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale codec system during the deserialization of game assets. The protected constructor strongly discourages direct manual instantiation. Game logic should not use `new MovementEffects()`.
- **Scope:** The lifetime of a MovementEffects object is bound to the component that owns it, such as a temporary status effect on an entity or a persistent zone attribute. It is a short-lived, contextual object.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup once its owning component is removed and no other references to it exist. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The internal state is mutable and consists of a series of boolean flags. A critical piece of logic is embedded in the CODEC's `afterDecode` hook: if the `disableAll` flag is true, all other restriction flags are forcibly set to true during deserialization. This ensures a consistent state after the object is created from data.
- **Thread Safety:** **This class is not thread-safe.** It contains no internal locking mechanisms. All interactions with an instance of MovementEffects must be confined to a single thread, typically the main server game loop or an entity's specific update tick. Unsynchronized access from multiple threads will result in race conditions and undefined behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | O(N) | Static codec for serializing and deserializing instances from game data. |
| isDisableAll() | boolean | O(1) | Returns the state of the master disable flag. |
| toPacket() | MovementEffects | O(N) | Converts the object into its corresponding network protocol packet for client synchronization. |

## Integration Patterns

### Standard Usage
The intended use is declarative. A designer defines the movement effects within a larger asset file (e.g., a spell, a debuff, or a zone configuration). The engine's asset loader uses the static CODEC to instantiate and populate the object, which is then applied by game logic.

```java
// Example of how a system might apply a loaded effect to a player
// Note: The MovementEffects object "effects" would be loaded from an asset, not created here.

public void applyStunEffect(Player player, MovementEffects effects) {
    player.getStatusManager().addMovementEffects(effects);
    player.getNetworkHandler().syncMovementEffects(); // Triggers toPacket() internally
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new MovementEffects()`. This bypasses the crucial `afterDecode` logic defined in the CODEC, potentially leading to an object with an inconsistent state (e.g., `disableAll` is true but `disableJump` is false).
- **State Mutation After Creation:** While the fields are mutable, modifying them after the object has been created can break the assumptions of the systems using it. These objects are generally intended to be treated as immutable once loaded.
- **Asynchronous Access:** Never read or write to a MovementEffects instance from a separate thread without external synchronization managed by the owning system.

## Data Pipeline
The primary flow for this object is from a static data definition on disk to a network packet sent to the game client.

> Flow:
> Game Asset (e.g., JSON) -> Server Asset Loader -> **MovementEffects.CODEC** -> **MovementEffects Instance** -> Entity Component -> `toPacket()` -> Network Packet -> Client Input System

