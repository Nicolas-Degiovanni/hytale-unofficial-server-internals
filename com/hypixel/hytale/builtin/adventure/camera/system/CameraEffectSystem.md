---
description: Architectural reference for CameraEffectSystem
---

# CameraEffectSystem

**Package:** com.hypixel.hytale.builtin.adventure.camera.system
**Type:** Event-Driven System

## Definition
```java
// Signature
public class CameraEffectSystem extends DamageEventSystem {
```

## Architecture & Concepts
The CameraEffectSystem is a specialized, reactive component within the server's Entity Component System (ECS) framework. Its sole responsibility is to translate server-side damage events into client-side visual feedback, specifically camera shake. It acts as a critical bridge between the abstract concept of entity damage and the concrete player experience.

This system operates by subscribing to the damage pipeline. By extending DamageEventSystem and registering with the DamageModule's *inspect damage* group, it guarantees its execution as part of the standardized damage application process. It does not initiate actions on its own; it only reacts to them.

Its operational scope is narrowly defined by an ECS Query for entities possessing both a PlayerRef and an EntityStatMap component. This ensures it only processes damage events for entities that are actively controlled by a player and have a concept of health, preventing wasted computation on non-player entities or objects without stats.

## Lifecycle & Ownership
- **Creation:** The CameraEffectSystem is instantiated and registered by the DamageModule during the server's bootstrap sequence. It is not created on-demand or per-event. Its existence is tied to the lifecycle of the core DamageModule.
- **Scope:** A single instance of this system persists for the entire server session. It is managed and invoked by the ECS system scheduler.
- **Destruction:** The system is de-registered and becomes eligible for garbage collection during server shutdown when the DamageModule is torn down.

## Internal State & Concurrency
- **State:** The CameraEffectSystem is **entirely stateless**. It does not cache data or maintain any state between invocations of its handle method. All necessary information (the entity, the damage details, world configuration) is provided as arguments by the ECS framework for each specific event. This design enhances predictability and robustness.
- **Thread Safety:** This system is **not thread-safe** and must not be invoked concurrently. It is designed to be executed by the main server thread as part of the sequential, single-threaded ECS tick cycle. The ECS framework's execution model provides the necessary guarantees to prevent race conditions.

## API Surface
The primary public contract is the `handle` method, which is intended for framework invocation only. Other methods serve to configure its integration with the parent ECS scheduler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handle(index, chunk, store, buffer, damage) | void | O(1) | Processes a damage event for a single player entity. Calculates shake intensity based on damage amount versus max health and dispatches a network packet to the affected client. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. To trigger its logic, one simply applies damage to a player-controlled entity. The engine's DamageModule will automatically route the event through this system.

To customize the camera shake for a specific attack, attach a meta-object to the Damage instance before it is applied.

```java
// Example: Applying damage with a specific, non-default camera effect
Damage customDamage = new Damage(source, cause, amount);
int customEffectIndex = getMyCustomEffectIndex();

// Attach a meta-object to override the default effect lookup
customDamage.putMetaObject(Damage.CAMERA_EFFECT, new Damage.CameraEffect(customEffectIndex));

// Apply the damage; the CameraEffectSystem will be triggered automatically
damageableComponent.applyDamage(customDamage);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new CameraEffectSystem()`. The system's lifecycle is strictly managed by the DamageModule and the ECS framework. Manual instantiation will result in a non-functional object that is not registered to receive events.
- **Manual Invocation:** Do not call the `handle` method directly. This bypasses the ECS scheduler, the damage pipeline's state management, and query filtering. Doing so can lead to unpredictable behavior, state corruption, or server exceptions.

## Data Pipeline
The CameraEffectSystem sits late in the damage processing pipeline, acting as a translator from a gameplay event to a network packet.

> Flow:
> Game Logic creates `Damage` object -> `DamageModule` receives event -> ECS Scheduler enters `InspectDamageGroup` -> **CameraEffectSystem** is invoked for matching player entities -> System reads `EntityStatMap` and `Damage` amount -> Calculates intensity -> Creates `CameraShakePacket` -> `PlayerRef` component is used to dispatch packet to the client's network buffer.

