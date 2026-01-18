---
description: Architectural reference for CameraInteraction
---

# CameraInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Data Model / Configuration Object

## Definition
```java
// Signature
public class CameraInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The CameraInteraction class is a server-side data model that defines a specific manipulation of the player's camera. It is a concrete implementation of the more abstract SimpleInteraction, specializing in camera control such as changing perspective or initiating a cinematic action.

This class is a cornerstone of Hytale's data-driven design philosophy. It is not intended to be instantiated or manipulated directly in procedural game logic. Instead, instances are deserialized from game configuration files (e.g., JSON or HOCON) via the static **CODEC** field. This allows designers and developers to define complex camera behaviors as data, decoupling game mechanics from the core engine code.

Its primary architectural function is to act as a translator between a high-level, data-defined camera behavior and the low-level network protocol. During an interaction, the server-side Interaction Module uses a CameraInteraction instance to generate and configure a `com.hypixel.hytale.protocol.CameraInteraction` packet, which is then sent to the client for execution.

The class design explicitly requires synchronization with the client, as indicated by `needsRemoteSync` and `getWaitForDataFrom`. This implies that the server initiates the camera change but may wait for acknowledgment or state updates from the client before the interaction is considered complete, ensuring a consistent view state between server and client.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale **Codec** system during the server's bootstrap or asset loading phase. The static `CODEC` field is used to deserialize a data structure from a configuration file into a fully-formed CameraInteraction object. **WARNING:** Manual instantiation via `new CameraInteraction()` will result in an unconfigured and non-functional object.

- **Scope:** The object's scope is transient and bound to a single interaction event. When a player triggers an action that uses this specific camera behavior, the corresponding pre-loaded instance is retrieved and used by the Interaction Module. It does not persist beyond the duration of that interaction.

- **Destruction:** The object becomes eligible for garbage collection once the interaction completes and it is no longer referenced by the active `InteractionContext` or the server's Interaction Module.

## Internal State & Concurrency
- **State:** The object's state is effectively immutable after its initial creation via the codec. Fields such as `action`, `perspective`, and `cameraInteractionTime` are populated once during deserialization and are not designed to be modified at runtime. This ensures that a configured interaction is predictable and consistent every time it is triggered.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be owned and managed exclusively by the server's main game loop thread. All method calls, especially `tick0`, must be performed on this thread to prevent severe race conditions and state corruption within the `InteractionContext`.

## API Surface
The public contract of CameraInteraction is defined by its overrides of the SimpleInteraction base class, which are called by the Interaction Module.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generatePacket() | Interaction | O(1) | Factory method to create the raw network packet for this interaction. |
| configurePacket(Interaction) | void | O(1) | Populates the provided network packet with the configured state (perspective, action, etc.). |
| tick0(...) | void | O(1) | Executes a single tick of logic, primarily for synchronizing state with the client. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns **Client**, signaling that the server must wait for client data to progress the interaction. |
| needsRemoteSync() | boolean | O(1) | Returns **true**, indicating that this interaction requires network communication with the client. |

## Integration Patterns

### Standard Usage
A developer or designer does not typically write Java code to use this class. Instead, they define its properties in a data file. The system then loads and executes it. The following conceptual example shows how the *engine* would use a deserialized instance.

```java
// Within the server's InteractionModule...
// This object is retrieved from a registry after being deserialized from a config file.
CameraInteraction cameraCutscene = interactionRegistry.get("world_intro_cutscene");

// The module then drives the interaction's lifecycle on the main game thread.
// This is a system-level operation, not typical game logic code.
cameraCutscene.tick0(isFirstRun, deltaTime, type, context, cooldownHandler);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new CameraInteraction()`. This bypasses the data-driven `CODEC` system and results in an object with null fields, which will cause `NullPointerException`s when processed by the engine.
- **Asynchronous Execution:** Do not call `tick0` or other methods from a separate thread. All interaction logic must be synchronized with the main server tick to prevent state corruption.
- **Runtime State Mutation:** Avoid modifying the public or protected fields of a CameraInteraction instance after it has been loaded. This breaks the data-driven contract and can lead to unpredictable behavior and client-server desynchronization.

## Data Pipeline
The flow of data for a CameraInteraction is from static configuration on disk to a real-time camera update on the client.

> Flow:
> Game Config File (e.g., JSON) -> Server-side **CODEC** Deserializer -> **CameraInteraction Instance** -> Interaction Module (`tick0`) -> `configurePacket()` -> Network Packet (`com.hypixel.hytale.protocol.CameraInteraction`) -> Client Network Handler -> Client Camera System -> Rendered Frame
---

