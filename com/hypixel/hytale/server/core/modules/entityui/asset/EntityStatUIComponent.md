---
description: Architectural reference for EntityStatUIComponent
---

# EntityStatUIComponent

**Package:** com.hypixel.hytale.server.core.modules.entityui.asset
**Type:** Configuration Asset

## Definition
```java
// Signature
public class EntityStatUIComponent extends EntityUIComponent {
```

## Architecture & Concepts
The EntityStatUIComponent is a server-side data-definition class that represents a single UI element designed to display a specific entity statistic, such as health, mana, or stamina. It is not a live, interactive component but rather a static template loaded from configuration files at server startup.

Its primary architectural role is to serve as a bridge between human-readable asset definitions (e.g., a string "health" in a JSON file) and the engine's optimized, network-ready representation (an integer index). This translation is handled by its static CODEC, which is the cornerstone of its design. The CODEC deserializes the configuration, validates it against the master list of defined EntityStatTypes, and populates the internal state.

During gameplay, the server uses these pre-loaded asset instances to generate network packets that instruct the client to render the corresponding UI element for an entity.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the server's asset loading system via the static CODEC field. This process occurs during server initialization when entity configuration files are parsed. Manual instantiation is an anti-pattern and will result in a non-functional component.
- **Scope:** An instance of EntityStatUIComponent is a singleton for its specific definition. It is loaded once and persists in the server's asset registry for the entire duration of the server session. It is treated as a read-only template.
- **Destruction:** The object is marked for garbage collection when the server shuts down and the central asset registry is cleared.

## Internal State & Concurrency
- **State:** The internal state is effectively immutable. The fields *entityStat* (string) and *entityStatIndex* (integer) are populated once during the deserialization and decoding process. The *afterDecode* hook within the CODEC guarantees that the state is fully resolved before the asset is made available to the rest of the engine.
- **Thread Safety:** This class is inherently thread-safe. Because its state is established at load-time and never modified, instances can be safely read and used by multiple threads simultaneously without locks or other synchronization primitives. The primary runtime operation, generatePacket, is a pure read-only function.

## API Surface
The primary public contract is the static CODEC used by the asset system. The only significant runtime method is generatePacket.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generatePacket() | com.hypixel.hytale.protocol.EntityUIComponent | O(1) | Constructs a network-ready packet from the component's internal state. This packet instructs the client to render a stat-based UI element. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly in Java code. Instead, it is defined declaratively within an entity's UI asset file (likely JSON or HOCON). The engine's asset loader handles the creation and registration.

```json
// Example pseudo-code from an entity asset file
{
  "ui": {
    "components": [
      {
        "type": "EntityStat",
        "EntityStat": "health" // This string is deserialized by the CODEC
      }
    ]
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new EntityStatUIComponent()`. Doing so bypasses the critical validation and index-resolution logic within the CODEC's `afterDecode` hook, resulting in an invalid `entityStatIndex` and subsequent runtime errors.
- **State Mutation:** Do not attempt to modify the `entityStat` or `entityStatIndex` fields after the object has been loaded. These assets are shared and treated as immutable templates.

## Data Pipeline
The primary function of this class is to transform a declarative configuration on disk into a binary network packet sent to the client.

> Flow:
> Configuration File -> Asset Loader -> **EntityStatUIComponent (via CODEC)** -> generatePacket() -> Network Packet -> Game Client

