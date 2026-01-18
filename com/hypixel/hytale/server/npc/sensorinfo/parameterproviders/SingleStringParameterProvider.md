---
description: Architectural reference for SingleStringParameterProvider
---

# SingleStringParameterProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo.parameterproviders
**Type:** Transient

## Definition
```java
// Signature
public class SingleStringParameterProvider extends SingleParameterProvider implements StringParameterProvider {
```

## Architecture & Concepts
The SingleStringParameterProvider is a fundamental, stateful component within the server-side NPC Artificial Intelligence framework. Specifically, it serves as a mutable data container for a single string value used by an NPC's "sensor" system.

Its primary architectural role is to act as a data conduit, decoupling the source of a parameter (e.g., an AI behavior tree node, an animation state) from its consumer (an AI sensor). A sensor can be configured to read from this provider without needing to know which system is responsible for populating the data. This class represents the simplest form of this pattern: a direct, settable value holder.

It inherits from SingleParameterProvider, indicating it belongs to a family of providers that manage a single parameter identified by an integer ID. It implements the StringParameterProvider interface, fulfilling the contract for supplying string-based data.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by higher-level AI components, typically during the configuration of an NPC's sensor suite. It is not a globally managed service and should not be treated as one.
- **Scope:** The lifetime of a SingleStringParameterProvider is tightly coupled to its owning sensor or AI behavior. It persists as long as that specific configuration is active for an NPC.
- **Destruction:** The object is eligible for garbage collection when the owning AI component is de-referenced. The `clear` method provides a mechanism for logical state reset, which is often called between AI ticks or when a sensor's state needs to be invalidated.

## Internal State & Concurrency
- **State:** The internal state is highly mutable, consisting of a single nullable String field. The core design is for this value to be frequently updated via `overrideString` and reset via `clear`. The default state upon creation or after a `clear` operation is null.

- **Thread Safety:** **This class is not thread-safe.** All methods perform direct, unsynchronized access to the internal `value` field. It is designed to be exclusively owned and operated by a single thread, which is typically the main server thread responsible for ticking the NPC's AI.

**WARNING:** Concurrent access from multiple threads will lead to race conditions and unpredictable behavior. Do not share instances of this class across different AI agents or threads without external synchronization, which is considered an anti-pattern.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getStringParameter() | String | O(1) | Retrieves the currently stored string value. Returns null if no value is set. |
| clear() | void | O(1) | Resets the internal state, setting the stored string value to null. |
| overrideString(String value) | void | O(1) | Sets or replaces the internal string value. |

## Integration Patterns

### Standard Usage
This provider is typically configured once and then updated and read from during the AI update loop. A controlling system sets the value, and a sensor system reads it within the same tick.

```java
// In an AI Behavior or setup logic:
// Assume 'provider' is a pre-configured SingleStringParameterProvider instance
provider.overrideString("HostilePlayer_123");

// In a Sensor's processing logic during the same tick:
String targetEntityName = provider.getStringParameter();
if (targetEntityName != null) {
    // Sensor logic uses the provided name
}
```

### Anti-Patterns (Do NOT do this)
- **State Persistence:** Do not rely on the value of this provider to persist across multiple AI ticks. Other systems may call `clear` or `overrideString` at any time. Always fetch the value when you need it.
- **Cross-Thread Modification:** Never modify or read an instance from a separate thread. The lack of synchronization makes this pattern inherently unsafe.
- **Null Assumption:** Do not assume `getStringParameter` will return a non-null value. The provider can be cleared at any time. Always perform a null check on the result.

## Data Pipeline
This class acts as a simple state-holding node within the larger NPC data flow, not as a processing stage itself.

> Flow:
> AI Behavior Tree Node -> `overrideString(value)` -> **SingleStringParameterProvider** (Holds `value`) -> Sensor Logic -> `getStringParameter()` -> AI Decision Making

