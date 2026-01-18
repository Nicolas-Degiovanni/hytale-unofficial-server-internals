---
description: Architectural reference for ExtraInfoProvider
---

# ExtraInfoProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo
**Type:** Contract

## Definition
```java
// Signature
public interface ExtraInfoProvider {
```

## Architecture & Concepts
The ExtraInfoProvider interface defines a contract for components that supply supplementary, type-safe data to the server's AI sensing systems. It is a core component of an extensible Strategy Pattern, allowing new types of contextual information to be attached to AI entities without modifying the core sensor logic.

In the Hytale AI architecture, sensors are responsible for gathering information about the world (e.g., sight, hearing). However, some AI behaviors require specialized data that is not part of the standard sensor payload. This interface provides a standardized mechanism for AI systems to query for this extra information in a decoupled manner.

The key architectural feature is the getType method. It uses the class of the implementation itself as a unique, type-safe key. This avoids fragile, string-based lookups or extensive use of instanceof checks, enabling a clean, registry-based retrieval system. An AI behavior can request a specific type of information, like HostilityInfo, and the system can efficiently locate the corresponding provider.

## Lifecycle & Ownership
As an interface, ExtraInfoProvider itself has no lifecycle. The lifecycle and ownership semantics apply to its concrete implementations.

- **Creation:** Implementations are typically instantiated alongside an AI entity's configuration or behavior tree. They are often created as part of a larger data structure that defines a sensor's capabilities.
- **Scope:** The lifetime of a provider instance is tied to the component that owns it, which is usually a specific sensor or an overarching AI controller for a single NPC. It may be short-lived (per-tick) or long-lived (per-NPC).
- **Destruction:** The object is eligible for garbage collection when its owning sensor or AI entity is destroyed. There are no explicit destruction methods defined in the contract.

## Internal State & Concurrency
The interface is stateless. However, the design imposes strict expectations on its implementations.

- **State:** Implementations should be treated as immutable data carriers. While not enforced by the interface, mutable state is strongly discouraged as it can introduce unpredictable behavior in the AI tick. If state must be calculated, it should be done at creation time.
- **Thread Safety:** Implementations must be thread-safe. The server's AI system may operate across multiple threads, and a single provider instance could theoretically be accessed concurrently. The safest approach is to ensure all implementations are fully immutable.

## API Surface
The public contract is minimal, focusing entirely on type identification.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | Class<? extends ExtraInfoProvider> | O(1) | Returns the Class object of the implementation. This acts as a unique key for lookup. |

## Integration Patterns

### Standard Usage
A developer creates a concrete class implementing this interface to represent a new piece of sensor data. This provider is then attached to an AI configuration, allowing other systems to query for it by its class type.

```java
// 1. Define a concrete provider
public class HostilityInfoProvider implements ExtraInfoProvider {
    private final boolean isHostile;

    public HostilityInfoProvider(boolean isHostile) {
        this.isHostile = isHostile;
    }

    public boolean isHostile() {
        return this.isHostile;
    }

    @Override
    public Class<? extends ExtraInfoProvider> getType() {
        // Return its own class as the key
        return HostilityInfoProvider.class;
    }
}

// 2. In an AI system, query for the specific provider
SensorSuite sensors = npc.getSensorSuite();
HostilityInfoProvider hostilityInfo = sensors.getExtraInfo(HostilityInfoProvider.class);

if (hostilityInfo != null && hostilityInfo.isHostile()) {
    // Initiate combat logic
}
```

### Anti-Patterns (Do NOT do this)
- **Violating The Type Contract:** The getType method must *always* return the class of the implementation itself. Returning a different class will break the lookup mechanism and lead to runtime failures or unpredictable behavior.
- **Returning Null:** The getType method must never return null. This would violate the contract and likely cause a NullPointerException in the sensor lookup system.
- **Mutable State:** Do not design implementations with mutable fields that are modified after construction. This can introduce severe race conditions and non-deterministic behavior in the AI system.

## Data Pipeline
This interface facilitates a data retrieval pattern rather than a continuous data flow. It allows the AI system to pull specific, strongly-typed information on demand.

> Flow:
> AI Behavior Tick -> Requires Hostility Data -> Sensor System -> Queries for **HostilityInfoProvider.class** -> Returns Instance -> AI Behavior Consumes Data

