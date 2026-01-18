---
description: Architectural reference for MultipleParameterProvider
---

# MultipleParameterProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo.parameterproviders
**Type:** Transient

## Definition
```java
// Signature
public class MultipleParameterProvider implements ParameterProvider {
```

## Architecture & Concepts
The MultipleParameterProvider is a structural component within the server-side NPC AI system. It implements the **Composite** design pattern, acting as a container that aggregates multiple child ParameterProvider instances. Its primary role is to serve as a multiplexer or a dispatcher for parameter lookups.

Instead of providing a parameter value directly, it holds a map of other ParameterProvider objects, each keyed by an integer. When queried with a specific parameter key, it delegates the request to the corresponding child provider in its collection. This architecture enables the creation of complex, hierarchical, and modular AI configurations. For instance, a single sensor can be configured to query different types of parameters by using a MultipleParameterProvider as its source, where each integer key represents a distinct parameter type (e.g., 1 for target distance, 2 for threat level).

This class is fundamental to decoupling AI sensors from the specific logic of how parameters are calculated, allowing for flexible and reusable AI components.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by higher-level AI configuration systems, such as an NPC behavior factory or a sensor loader. It is not managed by a central registry.
- **Scope:** The lifetime of a MultipleParameterProvider is tightly bound to its parent AI component (e.g., a specific sensor or behavior instance on an NPC). It persists as long as that component is active.
- **Destruction:** The object is eligible for garbage collection when its parent component is destroyed or reconfigured. The clear method is exposed to facilitate explicit cleanup of its children, often called during an AI state reset or NPC despawn sequence.

## Internal State & Concurrency
- **State:** The internal state is **mutable**. The primary state is the `providers` map, which is populated via the addParameterProvider method. This map can grow dynamically during the AI configuration phase.
- **Thread Safety:** This class is **not thread-safe**. The underlying Int2ObjectOpenHashMap is not synchronized. Concurrent calls to addParameterProvider can lead to map corruption. All population and modification of this object must occur in a single-threaded context, typically during the NPC initialization phase before it is active in the world. Read operations via getParameterProvider are safe only if no concurrent writes are occurring.

**WARNING:** Modifying a MultipleParameterProvider after it has been assigned to an active AI entity is an unsupported operation and will lead to unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getParameterProvider(int parameter) | ParameterProvider | O(1) | Retrieves a child provider associated with the given integer key. Returns null if no provider is mapped. |
| clear() | void | O(N) | Invokes the clear method on all contained child providers. N is the number of children. |
| addParameterProvider(int parameter, ParameterProvider provider) | void | O(1) | Maps a child provider to a specific integer key. Overwrites any existing provider at that key. |

## Integration Patterns

### Standard Usage
The intended pattern is to use this class as part of a builder or factory process. An instance is created, populated with all necessary child providers, and then passed as an immutable dependency to an AI component.

```java
// Example: Configuring a sensor during NPC initialization
MultipleParameterProvider rootProvider = new MultipleParameterProvider();

// Populate with specific providers for different parameters
rootProvider.addParameterProvider(SensorParameters.TARGET_DISTANCE, new TargetDistanceProvider(npc));
rootProvider.addParameterProvider(SensorParameters.HEALTH_PERCENT, new NpcHealthProvider(npc));

// Assign the fully configured provider to the sensor
someNpcSensor.setParameters(rootProvider);
```

### Anti-Patterns (Do NOT do this)
- **Runtime Modification:** Do not call addParameterProvider on an instance that is already in use by an active NPC. AI configuration should be treated as immutable during an AI tick.
- **Shared Instances:** Do not share a single MultipleParameterProvider instance across multiple, unrelated NPC entities if their parameter sources are meant to be different. This can lead to state corruption and unintended behavior.
- **Unbounded Growth:** Avoid dynamically adding providers to this class during gameplay. It is designed for initial setup, not as a dynamic runtime collection.

## Data Pipeline
This component acts as a routing step in a control flow rather than a data transformation pipeline. It directs a request to the correct downstream component.

> Flow:
> AI Sensor Request -> **MultipleParameterProvider.getParameterProvider(key)** -> Map Lookup -> Returned Child ParameterProvider -> AI Sensor uses Child Provider

