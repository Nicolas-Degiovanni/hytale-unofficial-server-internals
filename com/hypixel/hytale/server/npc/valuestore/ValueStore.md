---
description: Architectural reference for ValueStore
---

# ValueStore

**Package:** com.hypixel.hytale.server.npc.valuestore
**Type:** Component Data

## Definition
```java
// Signature
public class ValueStore implements Component<EntityStore> {
```

## Architecture & Concepts

The ValueStore is a performance-critical data component within the server-side Entity-Component-System (ECS) framework, specifically designed for storing arbitrary NPC (Non-Player Character) state. It functions as a raw, type-separated data container, optimized for low memory overhead and high-speed access.

Architecturally, ValueStore implements a variation of the **Flyweight** or **Prototype** pattern. A single, shared schema defined by the ValueStore.Builder is used to configure the layout of data slots for a specific *type* of NPC. Each individual NPC *instance* then receives its own lightweight ValueStore object, which contains only the primitive arrays for its state, sized according to the shared schema.

This design decouples the data's structure (the named slots) from its storage (the primitive arrays). The mapping from a semantic name like "health" to a raw integer slot is managed externally by the SlotMapper, which is held by the Builder. This allows the engine to manage thousands of NPC instances with minimal memory footprint, as the structural metadata is not duplicated for each entity.

Its primary role is to serve as the backing data store for higher-level systems like NPC behavior trees, scripting engines, and AI logic. These systems query the Builder for a slot index and then operate directly on an entity's ValueStore component.

## Lifecycle & Ownership

- **Creation:** A ValueStore is never instantiated directly. It is created exclusively by the `ValueStore.Builder.build()` method. This builder is typically configured once during the server's asset loading phase for each distinct NPC type. When a new NPC entity is spawned in the world, the entity creation logic calls `build()` to produce a fresh, empty ValueStore component to be attached to the new entity. The `clone()` method serves a similar purpose, creating a new, uninitialized store with the same dimensions.

- **Scope:** The lifecycle of a ValueStore instance is tightly bound to the lifecycle of the NPC entity it is attached to. It persists as long as the entity exists within the game world.

- **Destruction:** The ValueStore is marked for garbage collection when its parent NPC entity is destroyed and removed from the world's EntityStore. There is no explicit `destroy` or `cleanup` method; its memory is reclaimed by the JVM.

## Internal State & Concurrency

- **State:** The internal state is entirely **mutable**. It consists of three primitive arrays for strings, integers, and doubles. The state is considered "uninitialized" upon creation, with integer and double arrays pre-filled with sentinel values (Integer.MIN_VALUE and -Double.MAX_VALUE) to represent empty slots. This avoids the overhead of wrapper types like Optional.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locks or synchronization mechanisms. All read and write operations are direct, unsynchronized array accesses. Concurrency control is the responsibility of the calling context. In the Hytale server architecture, entity components are expected to be accessed only by the thread that owns the corresponding world or region, typically the main server tick loop.

**WARNING:** Unsynchronized access to a ValueStore instance from multiple threads will lead to race conditions, data corruption, and unpredictable server behavior. All interactions must be marshaled onto the appropriate game loop thread.

## API Surface

The public API is minimal, focusing exclusively on high-performance data manipulation via integer slot indices.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readString(int slot) | String | O(1) | Reads a String value from the specified slot. |
| storeString(int slot, String value) | void | O(1) | Writes a String value to the specified slot. |
| readInt(int slot) | int | O(1) | Reads an integer value from the specified slot. |
| storeInt(int slot, int value) | void | O(1) | Writes an integer value to the specified slot. |
| readDouble(int slot) | double | O(1) | Reads a double value from the specified slot. |
| storeDouble(int slot, double value) | void | O(1) | Writes a double value to the specified slot. |
| clone() | Component | O(N) | Creates a new, empty ValueStore with the same slot counts. N is the total number of slots. |

## Integration Patterns

### Standard Usage

The standard pattern involves retrieving an entity's ValueStore component and using pre-resolved slot indices to access its data. The slot indices themselves are obtained from a shared Builder instance associated with the NPC's type.

```java
// During NPC type definition (e.g., asset loading)
ValueStore.Builder npcTypeBuilder = new ValueStore.Builder();
int healthSlot = npcTypeBuilder.getIntSlot("health");
int targetIdSlot = npcTypeBuilder.getStringSlot("targetEntityId");

// During gameplay, within the server tick for a specific NPC entity
Entity npcEntity = ...;
ValueStore data = npcEntity.getComponent(ValueStore.getComponentType());

// Read current health
int currentHealth = data.readInt(healthSlot);

// Update target
data.storeString(targetIdSlot, "player-uuid-123");
```

### Anti-Patterns (Do NOT do this)

- **Direct Instantiation:** Never call `new ValueStore()`. The constructor is private and the component must be created via its corresponding `Builder` to ensure the internal arrays are correctly sized according to the NPC type's schema.

- **Cross-Type Access:** Do not use a slot index obtained from one `Builder` (e.g., for a "Zombie" type) to access the `ValueStore` of an entity of another type (e.g., a "Skeleton"). This will result in reading or overwriting incorrect data, leading to severe logic bugs.

- **Unsafe Concurrent Modification:** Do not read or write to a ValueStore from an asynchronous task or a different thread without explicit, external synchronization at the entity or world level.

## Data Pipeline

ValueStore does not process data in a pipeline; it is a terminal repository for an entity's state. Data flows *into* and *out of* it, driven by external game logic.

> **Write Flow:**
> Behavior Tree Node -> Decides to change state -> Resolves "property name" to `slot index` via Builder -> **ValueStore.storeInt(slot, value)**
>
> **Read Flow:**
> AI System -> Needs to check state -> Resolves "property name" to `slot index` via Builder -> **ValueStore.readInt(slot)** -> Returns `value` to AI System

