---
description: Architectural reference for InteractionConfiguration
---

# InteractionConfiguration

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config
**Type:** Configuration Object / DTO

## Definition
```java
// Signature
public class InteractionConfiguration implements NetworkSerializable<com.hypixel.hytale.protocol.InteractionConfiguration> {
```

## Architecture & Concepts

The InteractionConfiguration class is a data-centric object that defines the rules for player interactions associated with a specific item or entity. It is not a service or manager, but rather a container for properties that dictate behavior within the server's core Interaction Module.

Its primary architectural role is to decouple interaction logic from game asset definitions. Game designers can define complex interaction rules—such as reach distance, target highlighting, and action priorities—in external data files (e.g., JSON). The server loads these definitions at runtime using the powerful Hytale **Codec** system, which populates instances of this class.

The static **CODEC** field is the cornerstone of this design. It provides a robust, type-safe, and version-resilient mechanism for serialization and deserialization. This allows the underlying data format to change without requiring modifications to the game logic that consumes this configuration.

Furthermore, its implementation of the NetworkSerializable interface signifies a critical client-server integration point. The server serializes this configuration into a network packet and sends it to the client, ensuring that client-side predictions and visual feedback (like entity outlines) are perfectly synchronized with the server's authoritative state.

## Lifecycle & Ownership

-   **Creation:** Instances are almost exclusively instantiated by the Hytale **Codec** system during the server's asset loading phase. The `CODEC` reads a data source (like an item's JSON file) and constructs an InteractionConfiguration object. Direct instantiation via `new` is rare and typically reserved for creating static defaults like **DEFAULT** and **DEFAULT_WEAPON**.
-   **Scope:** The lifetime of an instance is bound to the asset it describes. It is loaded once when its parent asset (e.g., an item) is registered and persists in memory as a read-only template for the duration of the server's runtime.
-   **Destruction:** There is no explicit destruction method. Instances are managed by the Java Garbage Collector and are reclaimed when the assets they belong to are unloaded from memory.

## Internal State & Concurrency

-   **State:** Instances should be treated as **effectively immutable** after construction. While its fields are not declared final, the design pattern of loading from a static data source implies a read-only contract. The codec enforces this for some fields, such as wrapping the `useDistance` map in an unmodifiable view upon deserialization.
-   **Thread Safety:** This class is **conditionally thread-safe**. Because its state is intended to be immutable post-creation, it is safe for concurrent reads from multiple threads (e.g., different player threads accessing the same item configuration).

    **Warning:** Modifying the state of a shared InteractionConfiguration instance after it has been loaded is not thread-safe and will lead to unpredictable and difficult-to-diagnose bugs. This action violates the core design principles of the asset system.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPriorityFor(interactionType, slot) | int | O(1) | Calculates the interaction priority for a given type and equipment slot. Returns 0 if no specific priority is defined. |
| toPacket() | com.hypixel.hytale.protocol.InteractionConfiguration | O(N) | Serializes this object into its network-equivalent packet DTO. Complexity is proportional to the number of defined priorities. |

## Integration Patterns

### Standard Usage

Developers should not interact with this class directly. Instead, they should retrieve it from a higher-level system, such as an item definition, which has already been loaded and configured by the server's asset pipeline.

```java
// Pseudo-code demonstrating retrieval from a parent object
ItemDefinition heldItemDef = itemRegistry.getDefinition("my_sword");
InteractionConfiguration config = heldItemDef.getInteractionConfiguration();

// Use the configuration to make a logical decision
int priority = config.getPriorityFor(InteractionType.ATTACK, PrioritySlot.MAIN_HAND);
if (priority > 0) {
    // Proceed with attack logic
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Avoid using `new InteractionConfiguration()` in game logic. All interaction behavior should be data-driven and loaded from asset files via the `CODEC`. Manual creation bypasses the asset pipeline and creates brittle, hard-coded behavior.
-   **Post-Construction Mutation:** Never modify the fields of a cached or shared InteractionConfiguration instance. These objects are treated as read-only templates. Modifying one will affect every use of that item across the entire server.

## Data Pipeline

The InteractionConfiguration serves as a critical link in the data flow from static game assets to live client-server communication.

> Flow:
> Item Asset File (JSON) -> Server Asset Loader -> **Codec Deserialization** -> In-Memory **InteractionConfiguration** instance -> Player Logic queries instance -> Server calls `toPacket()` -> `protocol.InteractionConfiguration` -> Network Serialization -> Client


