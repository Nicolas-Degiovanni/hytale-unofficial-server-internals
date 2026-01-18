---
description: Architectural reference for UniqueClimateGenerator
---

# UniqueClimateGenerator

**Package:** com.hypixel.hytale.server.worldgen.climate
**Type:** Immutable Data Model

## Definition
```java
// Signature
public class UniqueClimateGenerator {
```

## Architecture & Concepts
The UniqueClimateGenerator is a specialized component within the server-side world generation pipeline responsible for the deterministic placement of unique, non-repeating climate zones. These zones represent significant, often hand-designed areas like special landmarks or quest hubs that must appear exactly once in a predictable location relative to other features or a parent origin.

The system operates on a two-phase model:

1.  **Definition (Template):** An initial instance is created containing an array of **Entry** records. Each Entry acts as a blueprint, defining a unique zone's properties, search constraints (e.g., min/max distance from an origin), and its relationship to a potential parent zone. This initial object is a template and cannot yet answer spatial queries.

2.  **Realization (Applied):** The **apply** method consumes the template instance, a world seed, and other climate context objects (**ClimateNoise**, **ClimateGraph**). It then orchestrates an asynchronous search process (**ClimateSearch**) to resolve the abstract definitions into concrete world coordinates. This operation produces a *new*, fully-realized instance where the **Unique** records are populated with the final calculated positions.

This two-phase, immutable pattern ensures that world generation logic is declarative and reproducible. The definitions can be loaded from configuration, while the realization is a pure function of the world seed and climate parameters.

## Lifecycle & Ownership
-   **Creation:** A "template" instance is typically instantiated by a higher-level world or dimension generator during its bootstrap phase. This is often driven by deserializing world configuration files that define the set of unique zones. The static **EMPTY** instance is used for worlds with no unique zones.

-   **Scope:** The initial template instance is short-lived. The realized instance, returned by the **apply** method, is a long-lived object. It is intended to be cached and held by the primary world or dimension context for the entire lifetime of the server session for that world.

-   **Destruction:** The object holds no native resources and is managed by the Java garbage collector. It is eligible for collection when the world it belongs to is unloaded.

## Internal State & Concurrency
-   **State:** This class is **immutable**. Its internal arrays, **entries** and **zones**, are final. The **apply** method does not mutate the object's state; it returns a new instance containing the results of the generation process. The state transitions from a definition-only form to a realized form containing calculated positions.

-   **Thread Safety:** The immutable design makes this class inherently thread-safe for read operations. However, its concurrent usage profile is complex and requires careful management.

    **WARNING:** The realized **Unique** records contain a **CompletableFuture** for the zone's position. The primary query method, **generate**, calls **position.join()**, which is a **blocking** operation. Calling **generate** on a newly-realized instance before its internal futures have completed will block the calling thread, potentially causing severe performance degradation in the chunk generation loop. The system that invokes the **apply** method is responsible for ensuring all returned futures are complete before passing the instance to downstream consumers like chunk generators.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generate(int x, int y) | int | O(N) | Returns the color of the unique zone at the given coordinates, or -1 if none. N is the number of unique zones. **WARNING:** This method blocks until the zone's position is computed. |
| getCandidates(Map) | Zone.UniqueCandidate[] | O(N) | Transforms the internal Entry definitions into a candidate array for use by other world generation systems. |
| apply(...) | UniqueClimateGenerator | O(N * M) | Asynchronously resolves zone positions. Returns a *new* instance with populated futures. This is a computationally expensive, non-blocking initiation call. |

## Integration Patterns

### Standard Usage
The correct usage pattern involves separating the asynchronous realization step from the synchronous query step. This is typically done during the world's initialization phase.

```java
// During world loading/initialization
UniqueClimateGenerator template = loadGeneratorFromConfig();
ClimateNoise noise = context.getService(ClimateNoise.class);
ClimateGraph graph = context.getService(ClimateGraph.class);

// 1. Asynchronously apply the template to get a realized instance
UniqueClimateGenerator realizedGenerator = template.apply(world.getSeed(), noise, graph);

// 2. Collect all position futures for completion handling
List<CompletableFuture<Vector2i>> allPositions = new ArrayList<>();
for (UniqueClimateGenerator.Unique zone : realizedGenerator.zones()) {
    allPositions.add(zone.position());
}

// 3. Wait for ALL unique zones to be placed before proceeding
CompletableFuture.allOf(allPositions.toArray(new CompletableFuture[0])).join();

// 4. Store the fully-baked generator for synchronous use in chunk generation
world.setUniqueClimateGenerator(realizedGenerator);

// Later, in a chunk generator (guaranteed to be safe and non-blocking)
int zoneColor = world.getUniqueClimateGenerator().generate(x, z);
```

### Anti-Patterns (Do NOT do this)
-   **Blocking in Loops:** Never call **generate** on an instance whose position futures have not been explicitly completed. Doing so inside a chunk generation loop will cause cascading world generation stalls.

    ```java
    // BAD: This will block and stall the server
    UniqueClimateGenerator gen = template.apply(seed, noise, graph);
    // The futures inside 'gen' are not complete yet!
    int color = gen.generate(x, y); // This will block!
    ```

-   **Direct Instantiation:** Do not use **new UniqueClimateGenerator()** in application logic. Instances should be created from configuration data by a central world generation service.

## Data Pipeline
The component transforms declarative rules into spatial data. The flow is unidirectional, moving from an abstract definition to a concrete, queryable state.

> Flow:
> World Configuration File -> Deserializer -> **UniqueClimateGenerator (Template)** -> apply() -> ClimateSearch -> **UniqueClimateGenerator (Realized, with pending Futures)** -> CompletableFuture.join() -> **UniqueClimateGenerator (Ready for query)** -> generate(x, y) -> Zone Color
---

