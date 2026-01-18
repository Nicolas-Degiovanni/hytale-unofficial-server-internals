---
description: Architectural reference for ApplicationEffects
---

# ApplicationEffects

**Package:** com.hypixel.hytale.server.core.asset.type.entityeffect.config
**Type:** Configuration Data Object

## Definition
```java
// Signature
public class ApplicationEffects implements NetworkSerializable<com.hypixel.hytale.protocol.ApplicationEffects> {
```

## Architecture & Concepts

The ApplicationEffects class is a server-side data container that defines the full spectrum of visual, auditory, and gameplay-modifying effects applied to an entity. It is not a service or manager; it is a passive data structure representing a configured asset.

Its primary architectural role is to serve as the deserialization target for entity effect definitions stored in asset files (e.g., JSON). The static `CODEC` field is the cornerstone of this design. It employs the Hytale `BuilderCodec` to declaratively map configuration keys from an asset file to the fields of this class. This codec-driven approach provides robust validation, property inheritance from parent assets, and a clean separation between asset definition and in-memory representation.

After being loaded and processed from an asset, an ApplicationEffects instance is then serialized into a network-optimized packet for transmission to the client. It therefore acts as a critical bridge between the server's asset configuration system and the client-server network protocol.

### Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the Hytale asset loading framework during server initialization. The public static `CODEC` field is invoked to parse an asset file and construct a corresponding ApplicationEffects object. Manual instantiation is an anti-pattern and should be avoided.
-   **Scope:** The object's lifetime is bound to the parent `EntityEffect` asset that defines it. These objects are typically cached by an asset manager and persist for the entire duration of the server session.
-   **Destruction:** Instances are eligible for garbage collection when the server's asset cache is cleared, which generally occurs during server shutdown.

## Internal State & Concurrency

-   **State:** The object is mutable during its creation and decoding phase. The `BuilderCodec` populates its fields directly from the asset data. After the `afterDecode` hook triggers the `processConfig` method, the object's state should be considered effectively immutable. It caches derived data, most notably converting string-based `soundEventId` values into more efficient integer `soundEventIndex` values for network transmission.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be instantiated and processed on the main server thread during the asset loading phase. Subsequent read-only access from multiple game threads is safe, but any modification after initial loading will lead to undefined behavior and state corruption.

## API Surface

The public API is minimal, reflecting its role as a data container. The primary interaction point is the serialization method for network transport.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.ApplicationEffects | O(N) | Serializes the object into a network packet. Complexity is linear to the number of configured particles. |
| getHorizontalSpeedMultiplier() | float | O(1) | Returns the configured speed modifier. |
| getKnockbackMultiplier() | float | O(1) | Returns the configured knockback resistance modifier. |

## Integration Patterns

### Standard Usage

Developers will not interact with this class directly. Instead, it is used implicitly by the game engine when an entity effect is applied. The system retrieves the configured `EntityEffect` asset, which contains an ApplicationEffects instance, and triggers its network serialization.

```java
// Hypothetical engine code demonstrating the flow
EntityEffect poisonEffect = AssetManager.get(EntityEffect.class, "hytale:poison");
ApplicationEffects effects = poisonEffect.getApplicationEffects();

// The engine serializes the effects into a packet to be sent to the client
Packet packet = effects.toPacket();
player.getConnection().send(packet);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ApplicationEffects()`. The object is uninitialized and invalid without being populated by the `BuilderCodec` from an asset file.
-   **Post-Load Modification:** Do not modify the public or protected fields of this class after it has been loaded by the asset system. Doing so will cause a state mismatch, as derived data like `soundEventIndexLocal` will not be recalculated.
-   **Incorrect Sound Event Handling:** Relying on the `soundEventId` fields during gameplay is inefficient. The engine uses the pre-calculated `soundEventIndex` fields, which are generated once in `processConfig`.

## Data Pipeline

The ApplicationEffects class is a key stage in the pipeline that transforms a static asset definition on disk into a live visual effect for a player.

> Flow:
> `EntityEffect.json` (Asset on Disk) -> Asset Loader -> **`BuilderCodec`** -> **ApplicationEffects** (In-Memory Object) -> `processConfig()` (ID to Index conversion) -> Game Logic Trigger -> `toPacket()` -> `protocol.ApplicationEffects` (Network Packet) -> Client Renderer

