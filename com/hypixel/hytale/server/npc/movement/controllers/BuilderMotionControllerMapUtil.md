---
description: Architectural reference for BuilderMotionControllerMapUtil
---

# BuilderMotionControllerMapUtil

**Package:** com.hypixel.hytale.server.npc.movement.controllers
**Type:** Static Utility / Type Token Provider

## Definition
```java
// Signature
public class BuilderMotionControllerMapUtil {
```

## Architecture & Concepts
The BuilderMotionControllerMapUtil class serves a single, highly specific purpose: to provide a static, compile-time, type-safe reference to the generic type `Map<String, MotionController>`. This class is a common Java pattern used to circumvent type erasure.

At runtime, the generic type parameters of a collection (e.g., String and MotionController) are normally lost. By instantiating a concrete object and exposing its Class object, this utility provides a "type token". This token can be used by reflection-based systems, such as a dependency injection framework or a service registry, to perform type-safe lookups and registrations of this specific map implementation.

It acts as a centralized, canonical definition for the map that stores all available motion controllers for NPCs, ensuring that any system component needing to access this map uses the exact same type definition.

### Lifecycle & Ownership
- **Creation:** The class is loaded by the JVM ClassLoader when first referenced. Its static fields are initialized once during this class-loading phase.
- **Scope:** As a class with only static members, its state persists for the entire lifetime of the application after being loaded.
- **Destruction:** The class and its static members are unloaded only when the corresponding ClassLoader is garbage collected, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** This class holds immutable static state. The internal `MAP_OBJECT_REFERENCE` is an empty, unmodifiable map used only for type inference. The public `CLASS_REFERENCE` is a `static final` field, making its reference immutable after initialization.
- **Thread Safety:** This class is inherently thread-safe. Its state is established during the JVM's class loading process, which is a synchronized operation. All subsequent reads of the `CLASS_REFERENCE` field from any thread are guaranteed to be safe and consistent without requiring any explicit locking.

## API Surface
The public contract of this class consists of a single static field. It has no methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CLASS_REFERENCE | Class<Map<String, MotionController>> | O(1) | Provides a static type token for a map of MotionController instances. This is the primary entry point for interacting with the utility. |

## Integration Patterns

### Standard Usage
This class is intended to be used as a key or token when interacting with a central registry or dependency injection container. The `CLASS_REFERENCE` is passed to the framework to request the shared instance of the motion controller map.

```java
// Example: Retrieving the shared map from a central service context
ServiceContext context = Server.getServiceContext();
Map<String, MotionController> motionControllers = context.getService(BuilderMotionControllerMapUtil.CLASS_REFERENCE);

// Use the map to configure an NPC's behavior
MotionController walkController = motionControllers.get("standard_walk");
npc.setMotionController(walkController);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class using `new BuilderMotionControllerMapUtil()`. It is a utility class designed for static access only. Instantiation provides no value and violates the design intent.
- **Reflection Abuse:** Do not use reflection to access or modify the private `MAP_OBJECT_REFERENCE`. This internal field is an implementation detail for creating the type token and is not intended for any other use. Modifying it can lead to undefined behavior.

## Data Pipeline
This class is not a participant in any data pipeline. It does not process, transform, or route data. Its role is strictly to provide a static type definition for use by other systems.

