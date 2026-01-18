---
description: Architectural reference for MetricResults
---

# MetricResults

**Package:** com.hypixel.hytale.metrics
**Type:** Transient Data Object

## Definition
```java
// Signature
public class MetricResults {
```

## Architecture & Concepts
The MetricResults class is an immutable container for arbitrary metrics data. It functions as a strongly-typed wrapper around a raw BSON document, providing a stable, read-only representation of a collected metrics payload.

This class is not a service or manager; it is a pure data structure. Its primary role is to act as a terminal object within the Hytale serialization framework, specifically for the metrics subsystem. The tight coupling with the Hytale Codec system, evidenced by the static CODEC and ARRAY_CODEC fields, indicates that its creation and serialization are managed exclusively by the framework.

By encapsulating a BsonDocument, it abstracts the underlying data format from consumers, ensuring that metric data is handled consistently and safely throughout the engine. The schema definition is intentionally generic (ObjectSchema), signifying that this container is designed to hold flexible and potentially evolving data structures without requiring code changes.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the internal MetricResultsCodec during the deserialization of a BSON or JSON data stream. Client code should never instantiate this class directly. The typical source is a network endpoint or a data file containing metrics information.
- **Scope:** Short-lived and transient. A MetricResults object exists only for the duration of processing a specific metrics payload. It is created, passed to a consumer system, and then becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or explicit cleanup methods. Ownership is transferred by reference, and the object is reclaimed once it is no longer reachable.

## Internal State & Concurrency
- **State:** Immutable. The internal BsonDocument is assigned once at construction and is declared final. No public or protected methods exist to modify the state of the object after it has been created.
- **Thread Safety:** This class is inherently thread-safe. Due to its immutability, a single MetricResults instance can be safely read by multiple threads concurrently without any risk of data corruption or race conditions. No external locking is required.

## API Surface
The public contract of this class is primarily exposed through its static Codec instances, which handle all serialization and deserialization. Direct interaction with instances is limited.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | static Codec | O(1) | Provides the canonical serialization and deserialization logic for a single MetricResults instance. |
| ARRAY_CODEC | static Codec | O(1) | Provides the serialization and deserialization logic for an array of MetricResults instances. |
| getBson() | protected BsonDocument | O(1) | **WARNING:** For internal framework use only. Retrieves the raw BsonDocument. Bypassing the Codec API is not supported. |

## Integration Patterns

### Standard Usage
Developers should never create or manage MetricResults instances directly. The standard pattern is to use the provided static CODEC to decode a data source into a MetricResults object for further read-only processing.

```java
// Assume 'bsonValue' is a BsonValue received from a network or database
// ExtraInfo.NONE is typically used unless a specific context is required.
MetricResults results = MetricResults.CODEC.decode(bsonValue, ExtraInfo.NONE);

if (results != null) {
    // The 'results' object is now a safe, read-only container.
    // Pass it to other systems that are designed to consume metrics data.
    metricsProcessor.process(results);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is protected for a reason. Do not use reflection or other means to call `new MetricResults()`. This bypasses the serialization framework and breaks the design contract.
- **State Modification:** Do not use reflection to modify the internal final bson field. The immutability of this class is a critical design guarantee, and violating it will lead to unpredictable and unstable system behavior.
- **Manual Serialization:** Do not attempt to serialize this object using standard Java serialization or other third-party libraries. Always use the provided static CODEC and ARRAY_CODEC to ensure compatibility with the Hytale engine.

## Data Pipeline
MetricResults represents a structured data snapshot at a specific point in the metrics data flow. It is the output of a decoding process and the input for a consumption process.

> Flow:
> Raw BSON/JSON Payload -> `MetricResults.CODEC.decode()` -> **MetricResults Instance** -> Metrics Consumer System

