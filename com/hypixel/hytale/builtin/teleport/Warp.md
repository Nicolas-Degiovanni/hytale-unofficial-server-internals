---
description: Architectural reference for Warp
---

# Warp

**Package:** com.hypixel.hytale.builtin.teleport
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
@Deprecated
public class Warp {
```

## Architecture & Concepts

The Warp class is a deprecated data structure that represents a persistent, named teleportation destination. It serves as a data container, encapsulating all necessary information to define a specific point in a world: a unique identifier, the target world's name, a precise Transform (position and rotation), the creator's identity, and a creation timestamp.

Its primary architectural feature is the static **CODEC** field. This complex, custom-built BSON codec defines the serialization and deserialization contract for Warp objects, enabling them to be persisted to and loaded from data stores like world files or a database. The codec's implementation reveals a history of data migration, with logic to handle both a legacy `Date` field and a current `CreationDate` field.

**WARNING:** This class is annotated as **Deprecated**. It represents a legacy approach to teleportation points. New development should avoid using this class and seek the modern equivalent, as this implementation relies on problematic patterns like static access to the global Universe state.

## Lifecycle & Ownership

-   **Creation:** A Warp instance is created in one of two ways:
    1.  **Deserialization:** The server's persistence layer uses the static `Warp.CODEC` or `Warp.ARRAY_CODEC` to instantiate Warp objects from a BSON data source. This is the most common creation path when loading world data.
    2.  **Direct Instantiation:** A new Warp is created programmatically via its constructor, typically by a server-side command or system responsible for managing warp points (e.g., a `/setwarp` command).

-   **Scope:** The lifetime of a Warp object is typically transient. It is loaded into memory, held by a manager service, used to facilitate a teleport, and then becomes eligible for garbage collection. It is not designed to be a long-lived, stateful service.

-   **Destruction:** As a simple DTO, Warp instances are managed by the Java Garbage Collector. There are no manual cleanup or disposal methods required.

## Internal State & Concurrency

-   **State:** The internal state of a Warp object is **Mutable**. Its fields are not final and are set either by the constructor or reflectively by the codec during deserialization.

-   **Thread Safety:** This class is **Not Thread-Safe**. It is a plain data container with no internal locking or synchronization. Concurrent modification of a Warp instance from multiple threads will lead to race conditions and undefined behavior.

    **WARNING:** The `toTeleport` method accesses global state via `Universe.get()`. Calling this method from asynchronous tasks or threads not synchronized with the main server tick could introduce severe concurrency issues. All interactions with this class should be confined to the main server thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | String | O(1) | Returns the unique, case-sensitive identifier for the warp. |
| getWorld() | String | O(1) | Returns the name of the world this warp belongs to. |
| getTransform() | Transform | O(1) | Returns the positional and rotational data. May be null. |
| toTeleport() | Teleport | O(N) | Converts the Warp into a runtime Teleport object. Involves a global world lookup by name, which can fail and return null if the world is not loaded. |

## Integration Patterns

### Standard Usage

The intended use is for a higher-level service to load Warp objects from a persistent source, retrieve one by its ID, and convert it into a Teleport object to be processed by the entity teleportation system.

```java
// Hypothetical WarpManager service
Warp targetWarp = warpManager.findWarpById("spawn");

if (targetWarp != null) {
    Teleport teleportAction = targetWarp.toTeleport();
    if (teleportAction != null) {
        playerEntity.teleport(teleportAction);
    } else {
        // Handle case where the warp's world is not loaded
        playerEntity.sendMessage("Error: Target world is not available.");
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **New Development:** Do not use this class in any new feature development. Its Deprecated status indicates it is superseded by a more robust system.
-   **Global State Reliance:** Do not call `toTeleport` and cache the resulting Teleport object. The underlying World instance could be unloaded, invalidating the cached object. The conversion should happen just-in-time.
-   **Ignoring Nulls:** The `toTeleport` method can and will return null if the target world is not loaded. Failure to check for this will result in NullPointerExceptions.
-   **Concurrent Access:** Do not share and modify a single Warp instance across multiple threads without external locking.

## Data Pipeline

The primary data flow for the Warp class is through the persistence layer.

> **Serialization (Saving a Warp):**
>
> `Warp` Object -> **Warp.CODEC**.encode() -> BSON Document -> World Save File / Database

> **Deserialization (Loading a Warp):**
>
> World Save File / Database -> BSON Document -> **Warp.CODEC**.decode() -> `Warp` Object

> **Runtime Usage:**
>
> `Warp` Object -> toTeleport() -> `Teleport` Object -> Entity Teleportation Module

