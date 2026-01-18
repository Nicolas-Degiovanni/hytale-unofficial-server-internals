---
description: Architectural reference for NPCAnimationSlot
---

# NPCAnimationSlot

**Package:** com.hypixel.hytale.server.npc.animations
**Type:** Enum

## Definition
```java
// Signature
@Deprecated
public enum NPCAnimationSlot implements Supplier<String> {
```

## Architecture & Concepts

**WARNING: This enum is deprecated and should not be used in new development. It is preserved for legacy compatibility and will be removed in a future version.**

The NPCAnimationSlot enum serves as a server-side abstraction and mapping layer for animation channels on a non-player character (NPC). Its primary architectural function is to decouple the server's internal game logic from the concrete data structures defined in the network protocol.

Each constant within this enum (Status, Action, Face) represents a logical slot where an animation can be played. It holds a direct reference to the corresponding `AnimationSlot` enum from the protocol layer. This adapter pattern ensures that if the underlying network protocol representation of an animation slot changes, the modification is isolated to this mapping enum, preventing cascading changes throughout the server's NPC animation systems.

Its deprecation indicates that this mapping is either no longer necessary or has been superseded by a more direct or flexible mechanism, likely by using the protocol-level `AnimationSlot` enum directly within server logic.

### Lifecycle & Ownership
- **Creation:** Instances are created and initialized by the Java Virtual Machine (JVM) during class loading. As an enum, its constants are static, final, and instantiated only once.
- **Scope:** Application-scoped. The enum constants persist for the entire lifetime of the server process.
- **Destruction:** Instances are destroyed when the application's class loader is garbage collected, which typically occurs at server shutdown.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant's internal state, specifically the `mappedSlot` field, is final and assigned during static initialization. It cannot be modified at runtime.
- **Thread Safety:** **Inherently thread-safe**. Due to its immutable nature, this enum can be safely accessed and shared across any number of threads without requiring external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | String | O(1) | Returns the name of the enum constant (e.g., "Action"). Fulfills the Supplier contract. |
| getMappedSlot() | AnimationSlot | O(1) | Returns the corresponding enum constant from the network protocol layer. This is the primary function of the class. |
| VALUES | NPCAnimationSlot[] | O(1) | A statically cached array of all enum constants. Provides a minor performance benefit over calling the `values()` method repeatedly. |

## Integration Patterns

### Standard Usage
The intended use is to translate the server-side slot concept into its network-protocol equivalent before serialization and transmission to the client.

```java
// Retrieve the protocol-level slot for network serialization
AnimationSlot protocolSlot = NPCAnimationSlot.Action.getMappedSlot();

// The protocolSlot is then used to build the network packet
AnimationPacket packet = new AnimationPacket(npcId, protocolSlot, animationId);
networkManager.send(packet);
```

### Anti-Patterns (Do NOT do this)
- **Current Usage:** Do not use this class in any new code. Its deprecated status signals that a new, preferred pattern exists for handling animation slots. You should use the protocol `AnimationSlot` directly or the replacement system.
- **Ordinal Reliance:** Do not rely on the `ordinal()` value of the enum for serialization or persistent storage. If the order of constants is changed in the future, all stored data will become corrupt. Use the `name()` or the `get()` method for a stable representation.

## Data Pipeline
This enum acts as a simple, in-memory transformation step when preparing NPC animation data for network transport. It translates a high-level server concept into a low-level protocol-defined value.

> Flow:
> Server-Side Animation System -> **NPCAnimationSlot** -> `getMappedSlot()` -> `AnimationSlot` (Protocol Enum) -> Network Packet Encoder -> Client

