---
description: Architectural reference for InfoProviderBase
---

# InfoProviderBase

**Package:** com.hypixel.hytale.server.npc.sensorinfo
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class InfoProviderBase implements InfoProvider {
```

## Architecture & Concepts
InfoProviderBase is a foundational component within the server-side NPC Artificial Intelligence framework. It serves as a generic, extensible container for various data sources that an AI sensor requires to perceive and interpret the game world.

This class acts as the central data hub for a given sensor. Its primary architectural role is to decouple the sensor's logic from the concrete sources of information. It implements a combination of the **Strategy** and **Registry** patterns, allowing a single sensor to be configured with multiple, specialized data providers at runtime.

There are three distinct categories of information it manages:
1.  **ParameterProvider:** A general-purpose provider, typically used for accessing configuration values or simple game state parameters.
2.  **ExtraInfoProvider (Registered):** A map of specialized, type-safe providers that are registered at construction. These represent persistent, well-known data sources for the sensor, such as faction alignment or target threat levels.
3.  **ExtraInfoProvider (Passed):** A single, transient provider that can be injected for a specific query or tick. This is used for highly contextual data that is not relevant for the sensor's entire lifecycle.

## Lifecycle & Ownership
-   **Creation:** As an abstract class, InfoProviderBase is never instantiated directly. Concrete subclasses are instantiated by AI factories or behavior tree nodes when an NPC's sensory system is initialized. The constructor is responsible for wiring up the persistent data providers (ParameterProvider and the map of ExtraInfoProviders).

-   **Scope:** The lifetime of a InfoProviderBase instance is tightly coupled to its owning AI sensor or behavior. It persists as long as the sensor is active, which may be for the entire life of the NPC or only for the duration of a specific AI state.

-   **Destruction:** The object is marked for garbage collection when its owning sensor is destroyed and all external references are cleared. It does not manage any native resources and has no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The internal state is **conditionally mutable**. The ParameterProvider and the map of ExtraInfoProviders are configured at construction and are treated as immutable for the object's lifetime. In contrast, the passedExtraInfo field is designed to be highly volatile and is mutated via the passExtraInfo method. This field holds transient, per-operation state.

-   **Thread Safety:** This class is **not thread-safe**. The mutable passedExtraInfo field creates a significant risk of race conditions if an instance is accessed by multiple threads concurrently.

    **WARNING:** All operations on an InfoProviderBase instance and its subclasses must be synchronized externally or confined to the single thread responsible for updating the owning NPC's AI.

## API Surface
The public API is designed for querying configured data sources.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getParameterProvider(int) | ParameterProvider | O(1) | Retrieves a parameter provider by its integer key. Returns null if not available. |
| getExtraInfo(Class) | E | O(1) | Retrieves a registered, persistent provider by its class type from the internal map. |
| passExtraInfo(E) | void | O(1) | Injects a transient, single-use provider. This overwrites any previously passed provider. |
| getPassedExtraInfo(Class) | E | O(1) | Retrieves the transient provider set by the last call to passExtraInfo. |

## Integration Patterns

### Standard Usage
A concrete sensor implementation receives a subclass of InfoProviderBase during its initialization. During its update tick, the sensor queries the provider for the specific data it needs to perform its logic.

```java
// Example from within a hypothetical "TargetingSensor" class
// The 'infoProvider' field is an instance of a concrete InfoProviderBase subclass.

public void findBestTarget() {
    // Retrieve a persistent, registered provider for faction data
    FactionInfoProvider factionInfo = this.infoProvider.getExtraInfo(FactionInfoProvider.class);
    if (factionInfo == null) {
        // Sensor is misconfigured, cannot proceed
        return;
    }

    for (Entity potentialTarget : getNearbyEntities()) {
        if (factionInfo.isHostile(potentialTarget)) {
            // ... process hostile target
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** It is a compile-time error to use `new InfoProviderBase()`. You must extend this class.
-   **Cross-Thread Sharing:** Never share a single InfoProviderBase instance across multiple AI update threads. The `passExtraInfo` method makes its state volatile and will lead to unpredictable behavior and data corruption.
-   **Registering Duplicate Providers:** The constructor will throw an `IllegalArgumentException` if you attempt to register two `ExtraInfoProvider` instances that implement the same core type. This is a system invariant.

## Data Pipeline
InfoProviderBase does not participate in a linear data pipeline. Instead, it functions as a data source or hub that is queried on-demand by an AI sensor.

> Flow:
> AI Sensor Update Tick -> Sensor Logic requests data -> **InfoProviderBase.getExtraInfo()** -> Delegates to specific registered provider -> Data returned to Sensor Logic

