---
description: Architectural reference for EasingConfig
---

# EasingConfig

**Package:** com.hypixel.hytale.builtin.adventure.camera.asset
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class EasingConfig implements NetworkSerializable<com.hypixel.hytale.protocol.EasingConfig> {
```

## Architecture & Concepts
The **EasingConfig** class is a fundamental data structure that defines the properties of a timed transition, specifically its duration and acceleration curve. It is not a service or manager, but rather a self-contained configuration object used extensively by systems that require smooth interpolation, such as the camera and animation systems.

Its primary role is to decouple the definition of an easing curve from the logic that executes it. This allows designers and developers to define complex camera behaviors and animations in data files (e.g., JSON or a proprietary binary format) which are then loaded and interpreted by the engine at runtime.

The static **CODEC** field is the cornerstone of this class's design. It provides a complete, self-contained definition for serialization and deserialization, including data validation rules. This makes **EasingConfig** a robust, data-driven component that can be reliably loaded from assets or transmitted over the network without requiring external mapping logic.

By implementing **NetworkSerializable**, this class explicitly declares its role as part of the data contract between the server and client, enabling the synchronization of any game state that relies on timed transitions.

### Lifecycle & Ownership
-   **Creation:** Instances are almost exclusively created by the engine's serialization systems using the provided static **CODEC**. This typically occurs when loading a parent asset (like a camera profile) from disk or when deserializing a network packet. Direct manual instantiation is rare and generally discouraged.
-   **Scope:** The lifetime of an **EasingConfig** instance is bound to its containing object. It is an ephemeral value object, not a persistent service. For example, if it is part of a camera profile, it will exist as long as that profile is held in memory.
-   **Destruction:** The object is managed by the Java garbage collector and is destroyed when it is no longer referenced. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** **EasingConfig** is a mutable Plain Old Java Object (POJO). Its internal fields, *time* and *type*, can be modified after creation. However, in practice, instances are typically treated as immutable after being deserialized.
-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization primitives. It is designed to be created, read, and used within a single, well-defined thread context, such as the main game thread or an asset loading thread.

**WARNING:** Sharing and concurrently modifying an **EasingConfig** instance across multiple threads will result in undefined behavior and data corruption.

## API Surface
The public contract is minimal, focusing on its role as a data holder and its network serialization capabilities.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | O(1) | **Static.** The self-describing codec for serialization and deserialization. |
| NONE | EasingConfig | O(1) | **Static.** A default, zero-time, linear easing configuration. |
| toPacket() | com.hypixel.hytale.protocol.EasingConfig | O(1) | Converts this asset-level object into its low-level network protocol equivalent. |

## Integration Patterns

### Standard Usage
**EasingConfig** is not typically used directly. Instead, it is retrieved from a parent configuration object that has been loaded by the asset system. The developer then passes this config object to a system that knows how to interpret it.

```java
// A system retrieves a pre-loaded asset containing camera data
CameraProfile cinematicProfile = assetManager.load("data/camera/cinematic_intro.asset");

// The EasingConfig is accessed from the parent profile
EasingConfig entryEasing = cinematicProfile.getEntryEasing();

// The config is then passed to a controller to execute the transition
cameraController.transitionTo(targetPosition, entryEasing);
```

### Anti-Patterns (Do NOT do this)
-   **Shared Mutable State:** Do not retrieve an **EasingConfig** from an asset and then modify its properties. This constitutes a modification of shared, cached state and will have unpredictable side effects throughout the application. If a custom configuration is needed, create a new instance.
-   **Manual Serialization:** Do not attempt to write custom serialization or deserialization logic for this object. The static **CODEC** is the single source of truth and contains critical validation logic (e.g., ensuring time is non-negative). Bypassing it can lead to invalid data entering the engine.

## Data Pipeline
**EasingConfig** is not a processing stage in a pipeline; it is the *data* that flows through it. It originates from a static data source and is used to configure a runtime process.

> **Asset Loading Flow:**
> Asset File (e.g., JSON) -> Engine Deserializer (using **EasingConfig.CODEC**) -> **EasingConfig Instance** (in memory) -> Camera System -> Interpolation Logic -> Final Camera Transform

> **Network Flow:**
> Network Packet -> Protocol Decoder -> `protocol.EasingConfig` -> Conversion via `toPacket` -> **EasingConfig Instance** -> Game Logic System -> State Update

