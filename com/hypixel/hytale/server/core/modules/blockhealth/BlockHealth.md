---
description: Architectural reference for BlockHealth
---

# BlockHealth

**Package:** com.hypixel.hytale.server.core.modules.blockhealth
**Type:** Transient Data Model

## Definition
```java
// Signature
public class BlockHealth implements Cloneable {
```

## Architecture & Concepts

BlockHealth is a fundamental data model representing the current durability and damage history of a single block within the game world. It is not a service or manager, but rather a lightweight state container designed for high-frequency instantiation and mutation.

Its primary role is to track the progressive damage a block sustains before it is destroyed. Each instance encapsulates two critical pieces of information: the remaining health as a float (where 1.0 is full health and 0.0 is destroyed) and the timestamp of the last damaging event.

A key architectural feature is the static `NO_DAMAGE_INSTANCE`. This is an immutable singleton object representing the state of a pristine, undamaged block. By using this shared instance as the default state for all blocks in the world, the engine avoids allocating millions of individual BlockHealth objects, resulting in a significant memory optimization. A new BlockHealth object is only created and associated with a block's coordinates the moment it first receives damage. This is an implementation of the Flyweight design pattern.

The presence of `serialize` and `deserialize` methods indicates its direct involvement in the server's data persistence and network synchronization pipelines, used for saving chunk data and transmitting block damage states to clients.

## Lifecycle & Ownership

-   **Creation:** A BlockHealth instance is lazily instantiated by a higher-level world management system (e.g., a `Chunk` or `World` service) at the exact moment a previously undamaged block receives damage. The default constructor is typically used, which initializes the block to full health.
-   **Scope:** The object's lifetime is strictly tied to the damaged state of the block it represents. It persists in memory as long as the block's health is below 1.0 but greater than 0.0.
-   **Destruction:** An instance becomes eligible for garbage collection under two conditions:
    1.  The block is completely destroyed (health reaches zero).
    2.  The block is fully repaired (health returns to 1.0).
    In both scenarios, the world management system will remove its reference to the BlockHealth object, effectively reverting the block's state to the shared `NO_DAMAGE_INSTANCE`.

## Internal State & Concurrency

-   **State:** The state of a BlockHealth object is **highly mutable**. Its core purpose is to have its `health` and `lastDamageGameTime` fields updated frequently by game logic. It is a self-contained Plain Old Java Object (POJO) with no external dependencies. The exception is the `NO_DAMAGE_INSTANCE`, which is strictly immutable and will throw an `UnsupportedOperationException` if mutation is attempted.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization primitives. All read and write operations on a BlockHealth instance must be externally synchronized. It is imperative that all modifications are performed exclusively on the main server game thread to prevent race conditions, data corruption, and non-deterministic behavior. For analysis or processing on other threads, a defensive copy should be created using the `clone` method.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isDestroyed() | boolean | O(1) | Returns true if health is at or below zero. This is the primary check for block destruction logic. |
| isFullHealth() | boolean | O(1) | Returns true if health is at or above 1.0. Used to determine if the object can be garbage collected. |
| serialize(ByteBuf) | void | O(1) | Writes the object's state (health, time) to a Netty byte buffer for network transport or persistence. |
| deserialize(ByteBuf, byte) | void | O(1) | Populates the object's state from a Netty byte buffer. The version parameter allows for future data format changes. |
| clone() | BlockHealth | O(1) | Creates a new BlockHealth instance with an identical state. Useful for creating safe copies for concurrent processing. |

## Integration Patterns

### Standard Usage

The intended use is for a world or chunk data structure to maintain a map of block positions to BlockHealth instances. This map only contains entries for blocks that are currently damaged.

```java
// Pseudo-code for a world damage system
Map<BlockPosition, BlockHealth> damagedBlocks = world.getDamagedBlockMap();
BlockPosition targetPos = ...;

// On block damage event
BlockHealth health = damagedBlocks.get(targetPos);
if (health == null) {
    // First time this block is damaged, create a new state object
    health = new BlockHealth();
    damagedBlocks.put(targetPos, health);
}

// Apply damage
float newHealth = health.getHealth() - damageAmount;
health.setHealth(newHealth);
health.setLastDamageGameTime(Instant.now());

if (health.isDestroyed()) {
    world.destroyBlock(targetPos);
    damagedBlocks.remove(targetPos); // Critical cleanup step
}
```

### Anti-Patterns (Do NOT do this)

-   **Pre-emptive Allocation:** Do not create `new BlockHealth()` for every block in a chunk. This completely negates the memory optimization provided by the `NO_DAMAGE_INSTANCE` and will cause severe performance degradation. Only create an instance on the first hit.
-   **State Modification of NO_DAMAGE_INSTANCE:** Do not attempt to cast and modify the shared `NO_DAMAGE_INSTANCE`. It is immutable for a reason, and any such attempt will throw a runtime exception.
-   **Cross-Thread Mutation:** Never modify a BlockHealth object from a network thread, physics thread, or any thread other than the primary game logic thread that owns the corresponding world chunk.

## Data Pipeline

BlockHealth is a data payload that flows through several core server systems.

> **Damage Application Flow:**
> Player Action -> Server Event -> World Damage System -> **BlockHealth** (Creation or Mutation) -> Chunk State Update

> **Network Synchronization Flow (Server to Client):**
> Chunk State Update -> Network Packet Builder -> `serialize(buf)` -> **BlockHealth** data written to ByteBuf -> TCP/UDP Packet -> Client

> **World Persistence Flow (Saving):**
> Server Shutdown/Save Command -> World Serializer -> `serialize(buf)` -> **BlockHealth** data written to file -> Chunk File on Disk

