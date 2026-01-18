---
description: Architectural reference for ItemWeapon
---

# ItemWeapon

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Configuration Data Object

## Definition
```java
// Signature
public class ItemWeapon implements NetworkSerializable<com.hypixel.hytale.protocol.ItemWeapon> {
```

## Architecture & Concepts
The ItemWeapon class is a data container that represents the weapon-specific properties of an item, as defined in a server asset file. It is not a service or a manager; its sole purpose is to hold configuration data that has been deserialized from disk.

This class is a critical component of the **Asset Definition Pipeline**. Its static CODEC field provides the declarative mapping between the human-readable asset format (e.g., JSON) and this in-memory Java representation.

A key architectural pattern employed here is **post-deserialization resolution**. The class is first populated with raw, string-based identifiers from the asset file (e.g., *rawStatModifiers*). The CODEC's *afterDecode* hook then triggers a resolution step, converting these strings into highly optimized, integer-based identifiers used by the runtime engine (e.g., *statModifiers*). This provides the benefit of human-readable configuration files while ensuring maximum performance during gameplay calculations.

As an implementation of NetworkSerializable, this class also defines the contract for converting its server-side representation into a lightweight, network-optimized packet for transmission to game clients.

## Lifecycle & Ownership
-   **Creation:** ItemWeapon instances are created exclusively by the server's asset loading system via the static CODEC. The process involves deserializing an item configuration file from disk. Direct instantiation is an invalid use case and will result in a non-functional object.
-   **Scope:** An instance of ItemWeapon has its lifetime tied to the parent item asset it belongs to. It is loaded once during server bootstrap (or asset hot-reloading) and persists in an asset registry for the entire server session.
-   **Destruction:** The object is eligible for garbage collection when the server's asset registry is cleared, typically during server shutdown.

## Internal State & Concurrency
-   **State:** The object is mutable only during the brief moment of its creation and deserialization by the CODEC. After the *afterDecode* hook completes, it becomes **effectively immutable**. The public API exposes no methods for altering its state.
-   **Thread Safety:** The object is **thread-safe for reads**. Because its state is fixed after initialization, multiple game threads (e.g., combat threads, player inventory threads) can safely access a shared ItemWeapon instance simultaneously without requiring locks or other synchronization primitives. The initial asset loading process is assumed to be single-threaded or otherwise synchronized by the asset management system.

## API Surface
The public API is minimal, focusing on read-only access to the resolved, game-ready data and network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getStatModifiers() | Int2ObjectMap | O(1) | Returns the resolved map of stat modifiers. Keys are integer IDs for high-performance lookups. |
| getEntityStatsToClear() | int[] | O(1) | Returns the resolved array of stat IDs that this weapon should clear upon application. |
| toPacket() | com.hypixel.hytale.protocol.ItemWeapon | O(N) | Creates a network-optimized packet. Complexity is proportional to the number of stat modifiers. |

## Integration Patterns

### Standard Usage
An ItemWeapon object should never be sought out directly. It is always accessed as a component of a parent item asset. Game logic, such as a combat system, retrieves the component to apply its effects.

```java
// Example from a hypothetical CombatSystem
void applyWeaponEffects(Entity attacker, Item equippedItem) {
    // Retrieve the weapon component from the generic item asset
    ItemWeapon weaponConfig = equippedItem.getComponent(ItemWeapon.class);

    if (weaponConfig != null) {
        // Access the resolved, high-performance stat data
        Int2ObjectMap<StaticModifier[]> modifiers = weaponConfig.getStatModifiers();
        
        // Apply modifiers to the attacker's stats
        EntityStatsModule.applyModifiers(attacker, modifiers);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ItemWeapon()`. This bypasses the CODEC deserialization and the critical *afterDecode* logic, leaving the object in an uninitialized state. All internal fields, especially the resolved stat maps, will be null.
-   **Accessing Raw Fields:** Avoid accessing the *rawStatModifiers* or *rawEntityStatsToClear* fields, even within the same package. They are intermediate representations used only during deserialization and are not guaranteed to be valid or complete after the object is constructed.
-   **Post-Creation Modification:** Do not use reflection or other means to modify the state of an ItemWeapon object after it has been loaded. The server's systems operate on the assumption that this data is immutable. Unpredictable behavior, from calculation errors to server crashes, may result.

## Data Pipeline
The ItemWeapon class is a key stage in the transformation of data from a human-readable configuration file into a network-ready packet.

> Flow:
> Item Asset File (JSON) -> Asset Manager -> **ItemWeapon.CODEC** -> **ItemWeapon Instance** (with resolved IDs) -> Combat System -> **toPacket()** -> Network Packet -> Game Client

