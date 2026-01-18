---
description: Architectural reference for ShopPageSupplier
---

# ShopPageSupplier

**Package:** com.hypixel.hytale.builtin.adventure.shop
**Type:** Data-Driven Factory

## Definition
```java
// Signature
public class ShopPageSupplier implements OpenCustomUIInteraction.CustomPageSupplier {
```

## Architecture & Concepts
The ShopPageSupplier is a specialized factory class that acts as a bridge between the server's generic interaction system and the specific adventure mode shop user interface. Its sole responsibility is to instantiate a ShopPage when a player triggers a corresponding interaction.

This class embodies the principle of configuration over code. Rather than hard-coding which shop opens for a given interaction, developers define this relationship in data files. The Hytale engine uses the provided **CODEC** to deserialize these configurations at runtime, creating an instance of ShopPageSupplier for each configured UI interaction.

This decouples the interaction trigger (e.g., an NPC, a sign, a trigger volume) from the UI implementation. The interaction system only needs to know that it holds a CustomPageSupplier; it does not need to know about the ShopPage, its dependencies, or the specific shopId it requires.

## Lifecycle & Ownership
- **Creation:** An instance of ShopPageSupplier is created by the Hytale Codec and serialization system when it parses an OpenCustomUIInteraction configuration from a game data file. It is never instantiated directly in code using the new keyword.
- **Scope:** The object's lifetime is bound to its parent OpenCustomUIInteraction configuration. It persists in memory as long as the interaction definition is loaded by the server.
- **Destruction:** The object is marked for garbage collection when the server unloads or hot-reloads the asset files containing the parent interaction configuration. There is no manual destruction method.

## Internal State & Concurrency
- **State:** The class holds a single piece of immutable state: the **shopId** string. This value is injected once during deserialization and is never modified. Consequently, the object is effectively immutable after its creation.
- **Thread Safety:** ShopPageSupplier is inherently thread-safe. Its immutable nature and lack of side effects in its factory method ensure that it can be safely accessed from any thread. In practice, it is primarily invoked by the server's main entity processing thread during interaction handling.

## API Surface
The public contract is defined entirely by the CustomPageSupplier interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tryCreate(ref, accessor, playerRef, context) | CustomUIPage | O(1) | Factory method that instantiates and returns a new ShopPage, configured with the internal shopId. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly via Java code. Instead, they configure it within a data file that defines an entity's interaction. The system then invokes the supplier internally.

The following pseudo-code illustrates how the **OpenCustomUIInteraction** class would use a configured supplier.

```java
// Conceptual example from within the interaction system
// A developer would NOT write this code.

// 'supplier' is the configured ShopPageSupplier instance
CustomPageSupplier supplier = this.getConfiguredSupplier();

// The system invokes tryCreate when a player interacts
CustomUIPage pageToShow = supplier.tryCreate(entityStoreRef, accessor, playerRef, context);

// The system then sends the page to the player
playerRef.getUser().getUIManager().open(pageToShow);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance with `new ShopPageSupplier()`. The object requires its `shopId` to be injected by the codec system. A manually created instance will have a null `shopId`, which will cause a NullPointerException when the resulting ShopPage is created.
- **Subclassing for Logic:** Avoid subclassing ShopPageSupplier to add complex logic. This class is intended to be a simple data container and factory. If custom logic is needed to determine a page, a different, more dynamic implementation of CustomPageSupplier should be created.

## Data Pipeline
The ShopPageSupplier is a critical link in the chain that translates a player's in-world action into a UI event.

> Flow:
> Player Interaction Event -> Server Interaction System -> OpenCustomUIInteraction Handler -> **ShopPageSupplier.tryCreate()** -> New ShopPage Instance -> Server Network Layer -> Client UI Manager -> Render Shop Interface

