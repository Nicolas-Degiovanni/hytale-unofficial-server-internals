---
description: Architectural reference for MarkerBlockState
---

# MarkerBlockState

**Package:** com.hypixel.hytale.server.core.universe.world.meta.state
**Type:** Interface

## Definition
```java
// Signature
public interface MarkerBlockState {
```

## Architecture & Concepts
The MarkerBlockState interface defines a contract for block state objects that can be logically associated with a world map marker. It serves as a behavioral marker, allowing disparate systems to identify and interact with block states that have a direct representation on the World Map.

This interface is a critical component of the server's world metadata system. Instead of the WorldMapManager maintaining a complex, bidirectional mapping of coordinates to markers, it delegates the storage of the marker reference directly to the block state itself. This pattern decouples the WorldMapManager from the specific implementation details of any given block, promoting a "capability-based" design. Any block state that needs to be represented on the map simply implements this interface.

The core responsibility of an implementing class is to accept and store a MarkerReference provided by an authoritative system, typically the WorldMapManager.

### Lifecycle & Ownership
As an interface, MarkerBlockState has no lifecycle itself. However, the lifecycle of any *implementing object* is strictly defined by the block it represents.

- **Creation:** An object implementing MarkerBlockState is instantiated when its corresponding block is created and its state needs to be tracked. This typically occurs during chunk generation, world editing, or when a player places a relevant block.
- **Scope:** The instance persists as long as the block exists in the world and maintains its marker-related state.
- **Destruction:** The object is eligible for garbage collection when the block is destroyed or its state changes to a type that no longer implements MarkerBlockState. The WorldMapManager is responsible for invalidating the associated marker *before* the state object is destroyed.

## Internal State & Concurrency
- **State:** Any class implementing this interface is inherently mutable. It is expected to hold a reference to a WorldMapManager.MarkerReference, which is injected post-instantiation.
- **Thread Safety:** Implementations of this interface are **not** guaranteed to be thread-safe. Block state can be modified by various server threads, including the main game loop, world generation workers, and network threads. Any system accessing or modifying the MarkerReference must do so within the synchronized context of the world or chunk update cycle.

**WARNING:** Unsynchronized access to a MarkerBlockState implementation from multiple threads will lead to race conditions and corrupted world map data. All modifications must be queued as part of a standard block update tick.

## API Surface
The public contract is minimal, consisting of a single method for dependency injection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setMarker(MarkerReference) | void | O(1) | Injects the world map marker reference into the block state. This method is intended to be called exactly once by the WorldMapManager during the block's initialization. |

## Integration Patterns

### Standard Usage
The primary interaction pattern involves a system checking if a generic block state object conforms to the MarkerBlockState contract, and if so, casting and interacting with it. This is a classic example of capability detection.

```java
// System-level code, likely within WorldMapManager
BlockState state = world.getBlockStateAt(position);

if (state instanceof MarkerBlockState) {
    MarkerReference newMarker = this.createMarkerForPosition(position);
    ((MarkerBlockState) state).setMarker(newMarker);
}
```

### Anti-Patterns (Do NOT do this)
- **External Invocation:** Do not call the setMarker method from any system other than the one responsible for managing world map markers. Doing so will desynchronize the block's state from the central marker registry.
- **Null Injection:** While not explicitly forbidden, passing a null MarkerReference to setMarker is discouraged. The removal of a marker should be handled by the WorldMapManager, which may involve replacing the block state entirely rather than nullifying a field.
- **State Assumption:** Do not assume a block at a given position implements MarkerBlockState. Always perform an `instanceof` check before casting.

## Data Pipeline
The data flow for this component is unidirectional, representing the injection of a reference from a manager into a specific state object.

> Flow:
> WorldMapManager (Marker Creation) -> **MarkerBlockState.setMarker(ref)** -> Internal field storage within the implementing block state object

