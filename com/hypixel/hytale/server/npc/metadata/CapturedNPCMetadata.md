---
description: Architectural reference for CapturedNPCMetadata
---

# CapturedNPCMetadata

**Package:** com.hypixel.hytale.server.npc.metadata
**Type:** Data Model / DTO

## Definition
```java
// Signature
public class CapturedNPCMetadata {
```

## Architecture & Concepts
CapturedNPCMetadata is a data-centric class that serves as a Plain Old Java Object (POJO) for storing state related to a captured Non-Player Character. Its primary architectural role is to act as a structured data container that can be reliably serialized and deserialized by the engine's core `Codec` system.

This class is not a service or a manager; it is pure data. It represents the specific metadata required to, for example, render an icon for a captured NPC in the UI, display its name, or restore its state from a save file. The tight integration with the `BuilderCodec` framework is the defining characteristic of this class, indicating it is a critical component of the game's data persistence and network synchronization layers. It is designed to be encoded into a larger data structure, such as player inventory or world entity data, before being written to disk or sent over the network.

## Lifecycle & Ownership
- **Creation:** Instances are primarily created by the Hytale `Codec` framework during the deserialization of game data. The static `CODEC` field provides the factory logic (`CapturedNPCMetadata::new`) for this process. Manual instantiation via `new CapturedNPCMetadata()` is also possible but is typically followed by immediate population of its fields.
- **Scope:** Ephemeral and context-dependent. The lifetime of a CapturedNPCMetadata object is strictly tied to the parent object that holds it, such as an item stack in an inventory. It does not persist globally or across sessions on its own.
- **Destruction:** The object is managed by the Java Garbage Collector. It is eligible for destruction as soon as it is no longer referenced by any active part of the game state.

## Internal State & Concurrency
- **State:** Mutable. This class is a standard data container with private fields and public getters/setters. Its state is intended to be populated once (either manually or via deserialization) and then read as needed. While modification after creation is possible, it is not the primary design pattern.
- **Thread Safety:** **Not thread-safe.** This object provides no internal synchronization mechanisms. It is a simple data structure and must not be modified or read by multiple threads concurrently without external locking. Unsynchronized access will result in race conditions and undefined behavior.

## API Surface
The public API consists of standard accessors for its data fields. The static `CODEC` and `KEYED_CODEC` fields are the primary entry points for serialization and deserialization operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getIconPath() / setIconPath(String) | String | O(1) | Manages the asset path to the UI icon for the captured NPC. |
| getRoleIndex() / setRoleIndex(int) | int | O(1) | Manages the numerical index representing the NPC's role or type. |
| getNpcNameKey() / setNpcNameKey(String) | String | O(1) | Manages the localization key for the NPC's display name. |
| getFullItemIcon() / setFullItemIcon(String) | String | O(1) | Manages the asset path to a larger, more detailed item icon. |

## Integration Patterns

### Standard Usage
The class is designed to be used implicitly by higher-level systems via the `Codec` framework. A system that needs to save or send data containing NPC metadata will use the provided static `KEYED_CODEC` to handle the transformation.

```java
// Example: Encoding metadata into a generic data container (e.g., NBT)
DataContainer container = new DataContainer();
CapturedNPCMetadata metadata = new CapturedNPCMetadata();
metadata.setNpcNameKey("mob.crawler");
metadata.setRoleIndex(5);

// The codec handles the complex serialization logic
CapturedNPCMetadata.KEYED_CODEC.encode(metadata, container);

// container now holds the serialized representation of the metadata
```

### Anti-Patterns (Do NOT do this)
- **Shared Mutable State:** Do not pass a single instance of CapturedNPCMetadata to multiple systems that may operate on different threads. Its mutable design makes it inherently unsafe for concurrent access without explicit, external synchronization.
- **Manual Serialization:** Avoid writing custom JSON, binary, or other serialization logic for this class. The static `CODEC` is the canonical source of truth for its data format and is designed to handle versioning and compatibility. Bypassing it will lead to data corruption and save-game incompatibility.

## Data Pipeline
This class is a payload within a larger data serialization pipeline. It represents a transformation point between an in-memory Java object and a serialized data format.

> **Serialization Flow:**
> In-Memory **CapturedNPCMetadata** Instance -> `KEYED_CODEC.encode()` -> Serialized Data Structure (e.g., NBT, JSON) -> Network Packet / Save File

> **Deserialization Flow:**
> Network Packet / Save File -> Serialized Data Structure -> `KEYED_CODEC.decode()` -> New In-Memory **CapturedNPCMetadata** Instance

