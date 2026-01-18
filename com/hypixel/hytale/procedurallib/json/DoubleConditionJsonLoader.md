---
description: Architectural reference for DoubleConditionJsonLoader
---

# DoubleConditionJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class DoubleConditionJsonLoader<K extends SeedResource> extends JsonLoader<K, IDoubleCondition> {
```

## Architecture & Concepts
The DoubleConditionJsonLoader is a specialized deserializer within the procedural generation framework. Its sole responsibility is to translate a specific JSON structure into a concrete, executable condition object that implements the IDoubleCondition interface. This class is a critical component of the engine's "configuration-driven design" philosophy, enabling developers and designers to define complex numeric-based logic in external JSON files without recompiling engine code.

The loader is designed with flexibility in mind, capable of interpreting multiple JSON formats:
1.  **Null or Missing:** Falls back to a predefined boolean default, representing an unconditional pass or fail.
2.  **Single Numeric Primitive:** Interpreted as a simple threshold, creating a SingleDoubleCondition (e.g., `value >= threshold`).
3.  **JSON Array:** Interpreted as a complex set of thresholds. In this case, the loader delegates parsing to a subordinate loader, DoubleThresholdJsonLoader, which produces a DoubleThresholdCondition.

This polymorphic parsing behavior allows for concise configuration for simple cases while supporting high-complexity scenarios through delegation.

## Lifecycle & Ownership
- **Creation:** An instance is created on-demand by a higher-level configuration parser when it encounters a JSON field that requires resolution into an IDoubleCondition. It is never managed as a long-lived service or singleton.
- **Scope:** The object's lifetime is exceptionally brief. It is designed to be single-use, existing only for the duration of the `load` method call.
- **Destruction:** The instance becomes eligible for garbage collection immediately after the `load` method returns the IDoubleCondition object. No manual resource management is necessary.

## Internal State & Concurrency
- **State:** The class is stateful but effectively **immutable** post-construction. All internal fields, including the source JsonElement and the fallback defaultValue, are final or treated as such. The `load` method is a pure function of this initial state.
- **Thread Safety:** This class is **thread-safe**. Due to its immutable nature, a single instance can be safely passed between threads, and the `load` method can be called concurrently without risk of data corruption. However, the standard usage pattern is to create a new instance for each parsing operation on a single thread.

## API Surface
The public contract is minimal, focused exclusively on the `load` operation. Constructors are for internal framework use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | IDoubleCondition | O(1) | Parses the internal JsonElement into a concrete IDoubleCondition. Throws NullPointerException if JSON is null and no defaultValue was provided. Throws Error on malformed JSON. |

## Integration Patterns

### Standard Usage
This loader is not intended for direct use by end-users. It is invoked by other parts of the configuration loading system. The typical pattern involves extracting a JsonElement from a larger document and passing it to this loader for specialized processing.

```java
// Assume 'parentJson' is a JsonObject being parsed by a higher-level system
JsonElement conditionJson = parentJson.get("heightCondition");
SeedString<MyResource> seed = ...;
Path dataFolder = ...;

// Instantiate the loader for a single, specific parsing task
DoubleConditionJsonLoader<MyResource> loader = new DoubleConditionJsonLoader<>(seed, dataFolder, conditionJson, false);

// Execute the load operation to get the final condition object
IDoubleCondition heightCondition = loader.load();

// The 'heightCondition' object is now used by the procedural engine
// The 'loader' instance is now discarded
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not hold a reference to a DoubleConditionJsonLoader instance to re-use it for parsing different JSON. Each instance is tied to the specific JsonElement provided at construction.
- **Null Default Value:** Constructing the loader with a null `defaultValue` is dangerous. If the source JSON is missing or null, the `load` method will throw a fatal NullPointerException. Always provide a safe fallback value.

## Data Pipeline
The DoubleConditionJsonLoader acts as a transformation step in the broader configuration-to-engine pipeline. It converts raw, declarative data into executable application logic.

> Flow:
> JSON Configuration File → GSON Parser → JsonElement → **DoubleConditionJsonLoader** → IDoubleCondition Object → Procedural Generation Engine

