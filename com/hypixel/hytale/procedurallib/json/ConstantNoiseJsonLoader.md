---
description: Architectural reference for ConstantNoiseJsonLoader
---

# ConstantNoiseJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient Factory

## Definition
```java
// Signature
public class ConstantNoiseJsonLoader<K extends SeedResource> extends JsonLoader<K, NoiseFunction> {
```

## Architecture & Concepts
The ConstantNoiseJsonLoader is a specialized, single-purpose factory component within the procedural generation library. Its sole responsibility is to deserialize a specific JSON object configuration into a fully realized ConstantNoise function instance.

This class is a concrete implementation of the abstract JsonLoader, forming part of a larger Strategy or Factory pattern. A higher-level orchestrator, responsible for parsing entire procedural configuration files, delegates the instantiation of specific noise function types to corresponding loader classes like this one. The system identifies the type of noise function required from the JSON structure and selects the appropriate loader to handle it.

The use of a generic SeedResource and the manipulation of a SeedString (`seed.append`) firmly places this loader within a deterministic, seed-based generation pipeline. It ensures that the resulting noise function is correctly integrated into the hierarchical seeding model of the engine.

### Lifecycle & Ownership
- **Creation:** An instance of ConstantNoiseJsonLoader is created on-demand by a parent factory or configuration parser. This parent system is responsible for providing the required constructor dependencies: the parent seed, the data folder context, and the specific JsonElement representing the constant noise configuration.
- **Scope:** The object's lifetime is ephemeral. It is designed to be instantiated, used for a single `load` operation, and then immediately discarded. Its scope is strictly confined to the method in which it is created.
- **Destruction:** The object holds no native resources and requires no explicit cleanup. It becomes eligible for garbage collection as soon as the `load` method returns and its reference goes out of scope.

## Internal State & Concurrency
- **State:** The internal state of the loader consists of the `seed`, `dataFolder`, and `json` element provided at construction. This state is treated as immutable for the object's short lifetime. The loader is fundamentally a stateful container for the parameters required to perform its single factory operation.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded execution within the procedural generation pipeline. The parent orchestrator is responsible for ensuring that each loader instance is confined to a single worker thread. No internal locking mechanisms are present.

## API Surface
The public contract is minimal, focused exclusively on the factory operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | NoiseFunction | O(1) | Deserializes the internal JSON state into a new ConstantNoise instance. This is the primary factory method. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by end-developers. It is invoked by the core procedural library's configuration parsing system. The pattern involves delegation from a generic parser to this specialized loader.

```java
// Pseudo-code demonstrating the expected invocation context
JsonElement noiseConfig = getNoiseConfigFromJson(file);
String noiseType = noiseConfig.get("type").getAsString();

NoiseFunction result;
if ("Constant".equals(noiseType)) {
    // The system creates a transient loader to handle this specific block
    ConstantNoiseJsonLoader loader = new ConstantNoiseJsonLoader(currentSeed, dataFolder, noiseConfig);
    result = loader.load();
}
// ... use the resulting NoiseFunction
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually construct this class with arbitrary JSON. It is designed to be instantiated by the engine's configuration loading system, which guarantees the context and JSON structure are valid.
- **Instance Caching:** Do not cache or reuse instances of ConstantNoiseJsonLoader. They are cheap to create and are intended to be single-use. Reusing an instance provides no performance benefit and violates the intended lifecycle.
- **Concurrent Access:** Never call the `load` method from multiple threads on the same instance. This will lead to unpredictable behavior and is an unsupported use case.

## Data Pipeline
The ConstantNoiseJsonLoader is a single, focused step in a larger data transformation pipeline that converts static configuration files into live procedural generation objects.

> Flow:
> WorldGen JSON File -> Master Configuration Parser -> JsonElement (for ConstantNoise) -> **ConstantNoiseJsonLoader** -> ConstantNoise Instance -> Procedural Engine Execution

