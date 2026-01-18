---
description: Architectural reference for PrefabBufferCall
---

# PrefabBufferCall

**Package:** com.hypixel.hytale.server.core.prefab.selection.buffer
**Type:** Transient Data Object

## Definition
```java
// Signature
public class PrefabBufferCall {
```

## Architecture & Concepts
The PrefabBufferCall class is a simple data structure, often referred to as a Parameter Object or Data Transfer Object (DTO). It does not contain any logic itself. Instead, its sole purpose is to encapsulate the contextual parameters required for a single, stateful operation within the server's prefab placement and world generation pipeline.

By bundling a source of randomness (Random) and a spatial orientation (PrefabRotation), it provides a clean, extensible contract for complex methods. This prevents method signatures from becoming cluttered with numerous parameters and allows new contextual information to be added in the future without breaking existing code.

This object is a critical component for ensuring deterministic and repeatable world generation. By explicitly passing a seeded Random instance, the system can guarantee that the same prefab selection and orientation will occur for a given set of initial conditions.

### Lifecycle & Ownership
-   **Creation:** Instantiated on-demand by a higher-level system, typically a world generator or a prefab placement service, immediately before a prefab selection or buffering operation is invoked.
-   **Scope:** Extremely short-lived. Its scope is confined to the duration of a single method call or a small, atomic transaction within the prefab system. It is almost always a stack-local object.
-   **Destruction:** Eligible for garbage collection as soon as the method that received it as a parameter completes. It holds no external resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** Highly mutable. Its fields are public and are intended for direct access. It is a plain data container with no internal state management or validation.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and consumed within a single thread of execution. Sharing a PrefabBufferCall instance across threads is a severe anti-pattern. The internal state of the contained Random object is not atomic, and concurrent access will lead to race conditions and destroy the deterministic nature of the generation process.

## API Surface
The public contract of this class consists of its constructors, as it has no methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PrefabBufferCall() | Constructor | O(1) | Creates an uninitialized instance. Fields will be null. |
| PrefabBufferCall(Random, PrefabRotation) | Constructor | O(1) | Creates a fully populated and ready-to-use instance. |

## Integration Patterns

### Standard Usage
A PrefabBufferCall should be created as a transient object to bundle parameters for a service call.

```java
// Assume a PrefabPlacementService and a seeded Random instance exist
PrefabPlacementService placementService = context.getService(PrefabPlacementService.class);
Random worldGenRandom = new Random(worldSeed);
PrefabRotation desiredRotation = PrefabRotation.NORTH;

// Create the call object to encapsulate the operation's context
PrefabBufferCall callContext = new PrefabBufferCall(worldGenRandom, desiredRotation);

// Pass the context object to the service for processing
placementService.selectAndBufferPrefab(targetZone, callContext);
```

### Anti-Patterns (Do NOT do this)
-   **Object Re-use:** Do not re-use a PrefabBufferCall instance across different, unrelated placement operations. The internal state of the Random object will be advanced, leading to non-deterministic results where determinism is expected. Always create a new instance per logical operation.
-   **Incomplete Initialization:** Do not use the default constructor and then pass the object to a service without populating its fields. This will result in a NullPointerException downstream.
-   **Long-Lived State:** Do not store a PrefabBufferCall as a field in a long-lived service or manager. It is a transient object and must be created per-operation to ensure thread safety and logical correctness.

## Data Pipeline
PrefabBufferCall does not process data; it *provides* contextual data to the pipeline. It acts as an input parameter that guides the behavior of subsequent stages.

> Flow:
> World Generation Service -> **PrefabBufferCall (Context Created)** -> Prefab Selection Logic -> Prefab Buffer Population

