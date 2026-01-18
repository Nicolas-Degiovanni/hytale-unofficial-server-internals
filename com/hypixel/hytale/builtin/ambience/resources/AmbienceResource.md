---
description: Architectural reference for AmbienceResource
---

# AmbienceResource

**Package:** com.hypixel.hytale.builtin.ambience.resources
**Type:** Component / Data Object

## Definition
```java
// Signature
public class AmbienceResource implements Resource<EntityStore> {
```

## Architecture & Concepts
The AmbienceResource is a data component that attaches to an EntityStore, which typically represents a discrete region of the game world. Its sole purpose is to hold state that overrides the default, procedurally selected ambient music for that specific region.

This class acts as a simple state container, not a service. It translates a human-readable music identifier string into an integer index, which is a direct reference into the global AmbienceFX asset map. This translation from string to integer is a performance optimization, preventing repeated string lookups by the core Ambience system.

The Ambience system polls this resource on its associated EntityStore. If present, it uses the forcedMusicIndex to play a specific track; otherwise, it falls back to its default selection logic.

## Lifecycle & Ownership
-   **Creation:** An AmbienceResource is not meant to be instantiated directly. It is created and attached to an EntityStore by a higher-level system, such as a world generation process, a zone controller, or a scripting event.
-   **Scope:** The lifecycle of an AmbienceResource is strictly bound to the lifecycle of the EntityStore it is attached to. It persists as long as its parent EntityStore is loaded in memory.
-   **Destruction:** The resource is marked for garbage collection when its parent EntityStore is unloaded and dereferenced. It has no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** The internal state is mutable, consisting of a single integer field, forcedMusicIndex. This field caches the integer index corresponding to a music asset ID, with a value of 0 representing "no override".
-   **Thread Safety:** This class is **not thread-safe**. The forcedMusicIndex field is accessed and modified without any synchronization primitives. All interactions with an instance of AmbienceResource must be performed on the main game thread or be externally synchronized by the calling system to prevent race conditions and memory visibility issues.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getResourceType() | static ResourceType | O(1) | Retrieves the global resource type definition for AmbienceResource. |
| setForcedMusicAmbience(String) | void | O(1) avg | Sets the music override. Translates the string ID to an integer index via the AmbienceFX asset map. A null ID resets the override. |
| getForcedMusicIndex() | int | O(1) | Returns the currently configured music override index. |
| clone() | Resource | O(1) | Creates a new instance with the same state. **Warning:** The current implementation is defective and returns null, which will cause a NullPointerException if used. |

## Integration Patterns

### Standard Usage
The intended pattern is to retrieve the resource from its parent EntityStore and then modify its state. The Ambience system will automatically detect and apply the change.

```java
// Retrieve the EntityStore for a specific world area
EntityStore worldZone = world.getZoneStorage(zoneCoordinates);

// Get the AmbienceResource, creating it if it does not exist
AmbienceResource ambience = worldZone.getOrCreateResource(AmbienceResource.getResourceType());

// Set the music for this zone
ambience.setForcedMusicAmbience("dungeon_theme_01");
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new AmbienceResource()`. The resource must be managed by its parent EntityStore to be recognized by the engine. Always use methods like `getOrCreateResource` on the EntityStore.
-   **Concurrent Modification:** Do not call `setForcedMusicAmbience` from a background or worker thread while the main thread might be reading it via `getForcedMusicIndex`. This will lead to undefined behavior.
-   **Relying on clone:** Do not use the `clone` method. Its implementation is faulty and will result in runtime exceptions.

## Data Pipeline
AmbienceResource serves as a data source for the Ambience system. It does not process data itself but holds state that influences a downstream pipeline.

> Flow:
> Game Logic (e.g., Quest Script) -> `setForcedMusicAmbience("id")` -> **AmbienceResource** (stores integer index) -> Ambience System (reads index during update tick) -> Audio Engine (schedules and plays the corresponding music track)

