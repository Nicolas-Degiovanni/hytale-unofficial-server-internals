---
description: Architectural reference for StatePair
---

# StatePair

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient

## Definition
```java
// Signature
public class StatePair {
```

## Architecture & Concepts
The StatePair class is an immutable data structure, functioning as a Value Object or Data Transfer Object (DTO). Its primary role within the server architecture is to create a canonical, type-safe representation of a single state within an NPC's state machine.

It acts as a bridge between human-readable state definitions found in asset files (e.g., "attack.heavy") and the highly optimized integer-based representations used by the server's core logic and network replication layers. By encapsulating the full string name, the primary state ID, and the sub-state ID into a single, immutable object, it ensures consistency and prevents data corruption during the asset loading and parsing pipeline.

This class is a fundamental component of the NPC asset builder system, not intended for direct use in high-level game logic.

## Lifecycle & Ownership
- **Creation:** Instances of StatePair are exclusively created during the NPC asset parsing phase. A higher-level component, such as an NpcAssetBuilder or StateMachineParser, instantiates this class after reading and interpreting state definitions from a source file (e.g., JSON or HOCON).
- **Scope:** The lifetime of a StatePair object is typically bound to the lifetime of the larger NPC configuration or asset object that contains it. It is a short-lived object during the parsing process, but once stored in a final configuration map, it persists as long as that NPC type is loaded in memory.
- **Destruction:** StatePair instances are managed by the Java Garbage Collector. They are eligible for cleanup when the parent NPC asset configuration is unloaded, such as during a server shutdown or a dynamic asset reload.

## Internal State & Concurrency
- **State:** The internal state is **Immutable**. All member fields are declared as final and are initialized only once within the constructor. This design guarantees that a StatePair object cannot be altered after its creation.
- **Thread Safety:** This class is inherently **Thread-Safe**. Due to its immutability, an instance of StatePair can be safely read by multiple threads concurrently without any risk of race conditions or data inconsistency. No external synchronization or locking mechanisms are required when accessing its data.

## API Surface
The public contract is minimal, consisting only of data accessors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFullStateName() | String | O(1) | Returns the complete, human-readable name of the state. |
| getState() | int | O(1) | Returns the primary integer identifier for the state. |
| getSubState() | int | O(1) | Returns the secondary integer identifier for the sub-state. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation by game logic developers. It is an internal product of the asset building system. A builder would typically create and store it in a map for later lookup.

```java
// Example from within a hypothetical asset parser
Map<String, StatePair> stateMap = new HashMap<>();
String stateName = "walking.normal";
int stateId = 1;
int subStateId = 0;

StatePair pair = new StatePair(stateName, stateId, subStateId);
stateMap.put(pair.getFullStateName(), pair);
```

### Anti-Patterns (Do NOT do this)
- **Ad-Hoc Creation:** Do not create new StatePair instances within game logic loops or per-entity updates. These objects are meant to be defined once during the server's asset loading phase and reused. Creating them on the fly defeats their purpose as canonical, shared definitions.
- **State Logic:** Do not attempt to add logic or behavior to this class. It is strictly a data container. State transitions and associated logic belong in the state machine controllers, not in this DTO.

## Data Pipeline
StatePair is a key artifact in the data transformation pipeline that converts declarative asset files into an executable in-memory representation of an NPC's behavior.

> Flow:
> NPC Asset File (JSON) -> Asset Parser -> **StatePair instance created** -> Stored in NpcStateMachineDefinition -> Referenced by NpcController for state transitions

