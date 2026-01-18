---
description: Architectural reference for MultiplePatternOp
---

# MultiplePatternOp

**Package:** com.hypixel.hytale.server.core.asset.type.tagpattern.config
**Type:** Data Transfer Object / Configuration Model

## Definition
```java
// Signature
public abstract class MultiplePatternOp extends TagPattern {
```

## Architecture & Concepts
MultiplePatternOp is an abstract base class that serves as a composite node within a hierarchical tag matching system. It is not a service or a manager, but rather a declarative configuration model representing a logical operation that groups multiple child TagPattern objects. Concrete implementations, such as an AND or OR operator, would extend this class.

Its primary role is to act as a bridge between server-side configuration assets and the network protocol. The static **CODEC** field is the central mechanism for this, enabling the Hytale codec system to deserialize structured data (e.g., JSON) from configuration files into a live object graph. This allows complex matching rules to be defined declaratively and loaded by the server at runtime.

The class embodies the Composite design pattern, where a MultiplePatternOp can contain other TagPattern objects, which themselves could be other MultiplePatternOp instances, forming a tree of conditions.

## Lifecycle & Ownership
- **Creation:** Instances of MultiplePatternOp subclasses are not created imperatively with the *new* keyword. They are instantiated exclusively by the Hytale **BuilderCodec** during the deserialization of server configuration assets. This process is typically initiated at server startup or during a configuration reload.
- **Scope:** The object's lifetime is bound to the server's loaded configuration. Once deserialized, it is effectively immutable and persists in memory for the duration of the server session or until the next configuration reload.
- **Destruction:** These objects are managed by the Java Garbage Collector. They are dereferenced and cleaned up when the root configuration object is discarded, typically on server shutdown or reload. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The core state is the protected **patterns** array, which holds the child TagPattern objects. This state is populated once during deserialization and is considered immutable thereafter. The class provides no public methods for modifying this array post-construction.
- **Thread Safety:** This class is inherently thread-safe. Its immutable-after-construction nature ensures that its state cannot be corrupted by concurrent access. The toPacket method is a pure function that only reads from the internal state, making it safe to be called from any thread without synchronization.

## API Surface
The public contract is minimal, focusing on data transformation rather than behavior.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.TagPattern | O(N) | Recursively transforms this configuration object and its N children into a network-ready protocol packet. This is the primary mechanism for synchronizing tag rules with the client. |

## Integration Patterns

### Standard Usage
This class is not intended for direct, imperative use in game logic. Its usage is declarative, defined within asset files. A system would retrieve a fully-formed TagPattern tree from a configuration manager and then use it.

```java
// A hypothetical system processes a pre-loaded configuration
// containing a MultiplePatternOp.
//
// NOTE: The 'rootPattern' is loaded and deserialized by the engine, not created here.
TagPattern rootPattern = someConfiguration.getTagMatchingRule();

// Convert the configuration model into a network packet for a client
com.hypixel.hytale.protocol.TagPattern packet = rootPattern.toPacket();
networkManager.sendPacket(player, packet);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never attempt to instantiate subclasses of MultiplePatternOp directly (e.g., `new AndPatternOp()`). The object graph must be constructed by the codec system from a valid configuration source to ensure correctness and immutability.
- **State Mutation:** Do not use reflection or other means to modify the internal **patterns** array after the object has been created. Doing so violates the class's design contract and can lead to unpredictable behavior and concurrency issues, as this configuration may be shared globally.

## Data Pipeline
MultiplePatternOp is a critical component in the configuration-to-network data pipeline for tag matching rules.

> Flow:
> Server Asset File (e.g., JSON) -> Hytale Codec System -> **MultiplePatternOp** (In-Memory Object) -> toPacket() -> Protocol DTO -> Network Encoder -> Client

