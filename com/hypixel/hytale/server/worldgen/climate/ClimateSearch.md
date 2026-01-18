---
description: Architectural reference for ClimateSearch
---

# ClimateSearch

**Package:** com.hypixel.hytale.server.worldgen.climate
**Type:** Utility

## Definition
```java
// Signature
public class ClimateSearch {
    // Note: This class is not designed to be instantiated.
    // All methods are static.
}
```

## Architecture & Concepts

The ClimateSearch utility is a foundational component of the server-side procedural world generation system. Its primary function is to perform targeted, asynchronous searches for world coordinates that satisfy a specific set of climatic conditions. This mechanism is critical for placing semantically important features, such as player spawn points, dungeons, or specific biome epicenters, where the environmental characteristics must be guaranteed.

Architecturally, ClimateSearch acts as a stateless computational service. It does not generate climate data itself; rather, it consumes pre-configured **ClimateNoise** and **ClimateGraph** objects, which represent the mathematical definition of the world's climate. The search is guided by a **Rule** object, which declaratively defines the desired outcome—for example, "a location that is temperate, not a deep ocean, and has low biome intensity".

The search algorithm is a deterministic, expanding circular probe. It samples points on the circumference of concentric circles, increasing the radius with each iteration. This pattern ensures a balance between search speed and coverage, allowing it to quickly find nearby matches before expanding to more distant and computationally expensive regions. The search can terminate early if a sufficiently high-quality match (defined by TARGET_SCORE) is found, optimizing performance for common cases.

## Lifecycle & Ownership

-   **Creation:** As a static utility class, ClimateSearch is never instantiated. Its methods are invoked directly on the class itself. The nested data structures, such as Rule and Range, are transient objects created by the calling system (e.g., the WorldGenerator) to define a specific search query.

-   **Scope:** This class is stateless and its methods are pure functions. The lifecycle of a search operation is encapsulated entirely within the `CompletableFuture` returned by the `search` method. This future, and the task it represents, lives until the search completes, fails, or is cancelled.

-   **Destruction:** No explicit destruction is required. The `Result` object and other transient data structures are eligible for garbage collection as soon as they are no longer referenced by the calling code.

## Internal State & Concurrency

-   **State:** ClimateSearch is entirely stateless. All necessary context, including the world seed, search parameters, and climate definitions, must be provided as arguments to the `search` method. All state related to an in-progress search (e.g., `bestScore`, `bestPosition`) is confined to local variables within the asynchronous task's lambda expression.

-   **Thread Safety:** This class is inherently thread-safe. The primary `search` method is non-blocking and immediately returns a `CompletableFuture`. The computationally intensive search loop is executed on a background thread from Java's common ForkJoinPool via `CompletableFuture.supplyAsync`. This design prevents the world generation process from stalling the main server thread while searching for a suitable location.

    **WARNING:** The `ClimateNoise` and `ClimateGraph` objects passed into the search must themselves be thread-safe, as they will be accessed from a worker thread.

## API Surface

The public contract is composed of the static `search` method and its associated data structures.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| search(seed, cx, cy, startRadius, searchRadius, rule, noise, graph) | CompletableFuture<Result> | O(N) | Asynchronously searches for a coordinate matching the climate Rule. Complexity is proportional to the area searched, which scales with the square of the searchRadius. |
| Rule | class | - | A data object defining the target climate parameters and their scoring weights. |
| Range | class | - | A helper object defining a target value, an acceptable radius, and a weight. |
| Result | record | - | An immutable record containing the position, score, and duration of a completed search. |

## Integration Patterns

### Standard Usage

The `search` method should be invoked by a higher-level world generation service. The returned `CompletableFuture` should be handled asynchronously to process the result without blocking the calling thread.

```java
// Standard asynchronous invocation from a world generation service
ClimateSearch.Rule spawnRule = new ClimateSearch.Rule(...);
CompletableFuture<ClimateSearch.Result> futureResult = ClimateSearch.search(
    worldSeed,
    0, 0,
    0, 5000,
    spawnRule,
    world.getClimateNoise(),
    world.getClimateGraph()
);

futureResult.thenAccept(result -> {
    if (result.score() > 0.5) {
        System.out.println("Found suitable spawn point: " + result.pretty());
        // Proceed with spawn point placement
    } else {
        System.err.println("Could not find a suitable spawn point.");
        // Handle fallback logic
    }
});
```

### Anti-Patterns (Do NOT do this)

-   **Blocking on the Future:** Never call `future.get()` on a critical thread, such as the main server tick loop. This will freeze the thread until the search completes, which can take hundreds of milliseconds, causing severe server lag. Always use `thenAccept`, `thenApply`, or other asynchronous chaining methods.

-   **Excessive Search Radius:** Invoking a search with `MAX_RADIUS` for routine operations is highly inefficient. This forces the system to evaluate a massive number of points, consuming significant CPU resources. Search radii should be constrained to the smallest reasonable value for the given task.

-   **Ignoring the Score:** The `Result` object contains a `score` field. It is a design error to assume that any non-zero score is acceptable. The calling logic must validate that the score meets its quality threshold before using the returned position. A low-score result indicates a poor match for the requested `Rule`.

## Data Pipeline

The flow of data for a single search operation is linear and self-contained. The utility transforms a declarative request into a concrete world coordinate.

> Flow:
> World Generator Service -> `ClimateSearch.Rule` (Input) -> **ClimateSearch.search()** -> Asynchronous Task -> Queries `ClimateNoise` & `ClimateGraph` -> `ClimateSearch.Result` (Output) -> World Generator Service Callback

