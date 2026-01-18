---
description: Architectural reference for UpdateWorldMapSettings
---

# UpdateWorldMapSettings

**Package:** com.hypixel.hytale.protocol.packets.worldmap
**Type:** Transient

## Definition
```java
// Signature
public class UpdateWorldMapSettings implements Packet {
```

## Architecture & Concepts

UpdateWorldMapSettings is a Data Transfer Object (DTO) that operates within the Hytale Network Protocol Layer. It is not a service or manager, but rather a structured message sent from the server to the client to configure the behavior and appearance of the in-game world map.

This packet's primary responsibility is to synchronize server-defined map rules with the client's UI. This includes enabling or disabling the map, controlling teleportation permissions, setting zoom limits, and providing biome-specific rendering data.

The binary layout of this packet follows a common engine pattern:
1.  **Nullable Bit Field:** A single byte at the start acts as a bitmask to indicate the presence of optional, variable-sized fields. This is a network optimization to avoid sending empty length prefixes for data that is not present.
2.  **Fixed-Size Block:** A 16-byte block containing simple, predictable data types like booleans and floats. This allows for extremely fast, non-branching deserialization of the core settings.
3.  **Variable-Size Block:** Following the fixed block, any variable-sized data structures, such as the biomeDataMap, are appended. Their presence is dictated by the initial null bit field.

This design prioritizes performance by minimizing payload size and allowing for rapid validation and deserialization on the client.

## Lifecycle & Ownership

-   **Creation:**
    -   **Server-Side:** Instantiated via its constructor (`new UpdateWorldMapSettings(...)`) by the server's game logic. It is populated with the current world's map configuration before being passed to a player's network channel for serialization.
    -   **Client-Side:** Instantiated exclusively by the network protocol layer through the static `deserialize` factory method. This occurs when an incoming network buffer is identified with Packet ID 240. Client game code should never create instances of this class directly.

-   **Scope:** This object is highly transient and its lifetime is bound to a single network message event. It exists only to be serialized, transmitted across the network, deserialized, and immediately consumed by a packet handler.

-   **Destruction:** The object is eligible for garbage collection as soon as the client-side packet handler completes its execution. No long-term references should be held to instances of this packet, as they represent a point-in-time snapshot of server settings.

## Internal State & Concurrency

-   **State:** The class is a **mutable** data container. All of its fields are public and can be modified after construction. This design is for performance and simplicity within the single-threaded context of a packet handler.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms. It is designed to be created, serialized, deserialized, and processed entirely within a single thread, such as a Netty I/O worker or the main client thread.

    **Warning:** Sharing an instance of UpdateWorldMapSettings across multiple threads without explicit external synchronization is unsafe and will result in data corruption and undefined behavior.

## API Surface

The public contract is dominated by static serialization and validation utilities, reflecting its role as a DTO.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static UpdateWorldMapSettings | O(N) | Constructs a new instance by reading from a ByteBuf. Throws ProtocolException on malformed data. N is the number of entries in the biome map. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the protocol specification. |
| computeSize() | int | O(N) | Calculates the exact number of bytes the object will consume when serialized. Used for pre-allocating buffers. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a lightweight, read-only check of a buffer to verify if it can contain a valid packet, without the cost of full object allocation. |
| clone() | UpdateWorldMapSettings | O(N) | Creates a deep copy of the packet, including a new HashMap for biome data. |

## Integration Patterns

### Standard Usage

A developer does not interact with this class directly. The network engine handles its lifecycle. The standard pattern involves a registered packet handler that receives the fully-formed object.

**Server-Side (Sending the Packet):**
```java
// In server logic, create and configure the packet
UpdateWorldMapSettings settings = new UpdateWorldMapSettings();
settings.enabled = true;
settings.allowTeleportToMarkers = world.getGameRule("allowMapTeleport");
settings.biomeDataMap = world.getBiomeColorData();

// Pass to the network system to send to a player
player.getConnection().sendPacket(settings);
```

**Client-Side (Handling the Packet):**
```java
// In a packet handler class, receive the deserialized object
public class WorldMapPacketHandler implements PacketHandler<UpdateWorldMapSettings> {
    private final WorldMapManager mapManager;

    @Override
    public void handle(UpdateWorldMapSettings packet) {
        // Apply the settings to the relevant game system
        this.mapManager.setEnabled(packet.enabled);
        this.mapManager.setZoomLimits(packet.minScale, packet.maxScale);
        this.mapManager.setTeleportationRules(packet.allowTeleportToCoordinates, packet.allowTeleportToMarkers);
        this.mapManager.updateBiomeData(packet.biomeDataMap);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Client-Side Instantiation:** Do not use `new UpdateWorldMapSettings()` on the client. The client's view of the map is strictly controlled by the server; it only receives and processes these packets.
-   **State Modification:** Do not modify a received packet instance. Treat it as a read-only snapshot of server state. If modification is necessary for some reason, use the `clone()` method first.
-   **Long-Term Storage:** Do not store references to this packet in long-lived services. The data within the packet should be copied into the state of the relevant system (e.g., a WorldMapManager), and the packet object itself should be discarded.

## Data Pipeline

The flow of this data originates from the server's authoritative game state and terminates with an update to the client's UI systems.

> **Flow:**
> Server Game State -> **new UpdateWorldMapSettings()** -> Serialization -> Network Transport -> Client Packet Dispatcher (ID 240) -> **UpdateWorldMapSettings.deserialize()** -> Packet Handler -> WorldMapManager -> UI Render Update

