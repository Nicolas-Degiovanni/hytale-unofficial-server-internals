---
description: Architectural reference for InteractionEffects
---

# InteractionEffects

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config
**Type:** Configuration Object

## Definition
```java
// Signature
public class InteractionEffects implements NetworkSerializable<com.hypixel.hytale.protocol.InteractionEffects> {
```

## Architecture & Concepts
The InteractionEffects class is a data-centric configuration object that defines the sensory feedback for an in-game interaction. It encapsulates all visual, auditory, and haptic effects—such as particles, sounds, and camera shake—that occur when a player or entity triggers an event.

Architecturally, this class serves two primary functions:

1.  **Asset Deserialization Target:** It is the direct target for deserializing effect definitions from game asset files (e.g., JSON). The static `CODEC` field employs a `BuilderCodec` to declaratively map configuration keys to class fields. This provides a robust, type-safe mechanism for content designers to specify complex effect combinations without writing code.

2.  **Server-to-Client Data Bridge:** It implements the `NetworkSerializable` interface, acting as a bridge between the server's configuration state and the client's runtime engine. The `toPacket` method transforms this high-level server configuration into a low-level, optimized network packet that the client can directly consume to render the specified effects.

This class is not a service or manager; it is a passive data container whose structure is fundamental to the engine's content pipeline.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the engine's asset loading system via the static `CODEC`. This process occurs once during the server's bootstrap phase as it parses all relevant game asset files. Direct instantiation is forbidden.
-   **Scope:** An instance of InteractionEffects, once loaded, persists for the entire server session. It is stored in memory as part of a larger configuration object (e.g., an item definition) and is treated as a shared, read-only template.
-   **Destruction:** Instances are garbage collected when the server shuts down and the central asset registries are cleared.

## Internal State & Concurrency
-   **State:** The state is **Effectively Immutable** after initialization. While the fields are not declared final, the object is mutated only once by the `processConfig` method immediately following deserialization. This post-processing step resolves string-based asset identifiers (e.g., `worldSoundEventId`) into more performant integer indices (e.g., `worldSoundEventIndex`).

-   **Thread Safety:** This class is **not thread-safe for mutation**. However, because all mutation is confined to the single-threaded server startup sequence, it is **completely safe for concurrent reads** during gameplay. The design assumes a "write-once, read-many" pattern, eliminating the need for locks or other synchronization primitives at runtime.

## API Surface
The public API is minimal, primarily exposing the transformation logic and read-only accessors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.InteractionEffects | O(N) | Serializes the object into a network packet for client consumption. N is the number of particles and trails. |
| processConfig() | void | O(1) | **Internal.** Resolves asset IDs into integer indices. Called automatically by the codec after deserialization. |

## Integration Patterns

### Standard Usage
This class is not intended for direct manipulation in gameplay logic. Instead, a higher-level system retrieves the pre-configured object and uses it to generate a network packet that is broadcast to relevant clients.

```java
// Example: A system handling item usage
public void onPlayerUseItem(Player player, Item item) {
    // Retrieve the pre-loaded effects configuration from the item's definition
    InteractionEffects effects = item.getDefinition().getOnUseEffects();

    if (effects != null) {
        // Transform the configuration into a network packet
        com.hypixel.hytale.protocol.InteractionEffects packet = effects.toPacket();

        // Broadcast the packet to clients who should see the effects
        world.broadcastPacket(packet, player.getPosition());
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Runtime Modification:** Never modify the fields of an InteractionEffects object after the server has started. These objects are shared templates, and runtime changes will lead to unpredictable behavior for all interactions using that template.
-   **Direct Instantiation:** The constructor is `protected` for a reason. Attempting to create an instance with `new InteractionEffects()` will result in an uninitialized object that bypasses the critical `processConfig` step, leaving asset indices unresolved.
-   **Relying on String IDs:** Do not use the string-based asset identifiers (e.g., `getWorldSoundEventId()`) in performance-critical runtime code. The engine is optimized to use the pre-calculated integer indices (e.g., `getWorldSoundEventIndex()`) which are sent in the network packet.

## Data Pipeline
The primary flow for this class is from a static configuration file on disk to a transient network packet sent to the game client.

> Flow:
> Game Asset (JSON) -> Server Asset Loader -> **InteractionEffects** (Instance with resolved indices) -> `toPacket()` -> Network Packet -> Client Rendering & Audio Engine

