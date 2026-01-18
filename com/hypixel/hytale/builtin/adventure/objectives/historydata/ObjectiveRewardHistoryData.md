---
description: Architectural reference for ObjectiveRewardHistoryData
---

# ObjectiveRewardHistoryData

**Package:** com.hypixel.hytale.builtin.adventure.objectives.historydata
**Type:** Data Model (Abstract Base)

## Definition
```java
// Signature
public abstract class ObjectiveRewardHistoryData {
```

## Architecture & Concepts
ObjectiveRewardHistoryData is an abstract base class that serves as the foundational data model for recording rewards granted to a player upon objective completion. It is a critical component of the adventure mode persistence layer, ensuring that the history of player achievements and their corresponding rewards can be reliably serialized, stored, and deserialized.

The core architectural pattern employed here is a **type-discriminator-based serialization system**, facilitated by the static `CODEC` field. This `CodecMapCodec` acts as a central registry for all concrete implementations of ObjectiveRewardHistoryData. When serializing, it injects a "Type" field into the output data, identifying the specific subclass. During deserialization, it reads this "Type" field first to determine which concrete codec to use for instantiating the correct object.

This design makes the system highly extensible. New reward types can be added to the game by simply creating a new subclass of ObjectiveRewardHistoryData and registering its specific codec with the central `CODEC` registry, without modifying the core objective history processing logic.

## Lifecycle & Ownership
- **Creation:** Instances of concrete subclasses are created in two scenarios:
    1. By the objective completion system at the moment a reward is granted to a player.
    2. By the `CODEC` during deserialization when loading player data from a save file or receiving it over the network.
- **Scope:** These are short-lived data transfer objects. Their lifetime is tied to the parent data container, such as a player profile or a session history object. They do not persist independently.
- **Destruction:** Instances are eligible for garbage collection as soon as the parent data structure that holds them is discarded. There is no manual destruction or cleanup required.

## Internal State & Concurrency
- **State:** This abstract class defines no state. Concrete implementations are intended to be **immutable** data containers. Once created, their fields, which represent the details of a past reward, should not change.
- **Thread Safety:** Instances of subclasses should be considered thread-safe for reads due to their immutable nature. The static `CODEC` field is a thread-safe singleton, designed to be accessed concurrently by serialization and deserialization workers.

## API Surface
As an abstract class, it presents no instantiable API. Its contract is defined by its role as a base class and its static `CODEC` field, which is the primary public interaction point for the serialization system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | CodecMapCodec | O(1) | The static, shared codec for serializing and deserializing any concrete subclass of ObjectiveRewardHistoryData. |
| BASE_CODEC | BuilderCodec | O(1) | A foundational codec used to build upon for concrete implementations. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. Instead, they define concrete subclasses representing specific reward types and register them with the serialization system.

```java
// 1. Define a concrete, immutable reward data class
public class XpRewardHistoryData extends ObjectiveRewardHistoryData {
    public static final Codec<XpRewardHistoryData> CODEC = ...; // Codec definition
    private final int amount;
    // Constructor, getters...
}

// 2. Register the new type with the central codec (typically during engine bootstrap)
ObjectiveRewardHistoryData.CODEC.register("XP", XpRewardHistoryData.CODEC);

// 3. The engine now automatically handles serialization/deserialization
// (This is framework-level code, not typical user code)
ObjectiveHistory history = ...;
history.addReward(new XpRewardHistoryData(100));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** It is a compile-time error to attempt `new ObjectiveRewardHistoryData()`. This class exists only to be extended.
- **Forgetting Registration:** Creating a concrete subclass without registering its codec with the static `CODEC` map will result in a `DeserializationException` when the system encounters that reward type in a save file. The system will not know how to reconstruct the object.
- **Mutable Subclasses:** Defining a subclass with mutable fields violates the design contract. These objects represent a historical record and must be immutable to prevent data corruption and ensure thread safety.

## Data Pipeline
This class is a passive data structure that sits within a larger data flow for persistence.

> **Serialization Flow:**
> Game Event (Objective Complete) -> Reward Processor -> **Concrete ObjectiveRewardHistoryData instance created** -> PlayerProfile -> Serializer using `CODEC` -> Binary/JSON Data -> Save File / Network Packet

> **Deserialization Flow:**
> Save File / Network Packet -> Binary/JSON Data -> Deserializer using `CODEC` -> **Concrete ObjectiveRewardHistoryData instance created** -> PlayerProfile -> Game State

