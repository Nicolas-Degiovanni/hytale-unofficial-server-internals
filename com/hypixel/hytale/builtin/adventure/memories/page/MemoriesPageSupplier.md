---
description: Architectural reference for MemoriesPageSupplier
---

# MemoriesPageSupplier

**Package:** com.hypixel.hytale.builtin.adventure.memories.page
**Type:** Utility

## Definition
```java
// Signature
public class MemoriesPageSupplier implements OpenCustomUIInteraction.CustomPageSupplier {
```

## Architecture & Concepts
The MemoriesPageSupplier is a stateless factory class that implements the CustomPageSupplier interface. Its sole responsibility is to instantiate a MemoriesPage object on demand.

This class serves as a crucial bridge within the server's **Interaction System**. It decouples the high-level definition of a player interaction (e.g., a player right-clicking a specific block) from the low-level code that constructs the resulting user interface. When an interaction is configured to open a custom UI, the system delegates the page creation task to a registered supplier like this one. This allows game designers to define complex UI-driven interactions in data files by simply referencing a supplier, without needing to modify core Java code.

It embodies the **Strategy Pattern**, where the algorithm for creating a UI page is encapsulated in a concrete implementation that can be selected at runtime by the parent interaction handler.

### Lifecycle & Ownership
- **Creation:** A single instance of MemoriesPageSupplier is likely created by the server's dependency injection or module system during server bootstrap. It is then registered with the Interaction System to be associated with specific interaction types.
- **Scope:** The object is a session-scoped singleton. It persists for the entire lifetime of the server process.
- **Destruction:** The instance is discarded and eligible for garbage collection only upon server shutdown.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields and its behavior is determined exclusively by the arguments passed to its methods. Each call to tryCreate produces a new, independent MemoriesPage instance.
- **Thread Safety:** MemoriesPageSupplier is inherently **thread-safe**. The absence of internal state means that it can be safely invoked by multiple server threads concurrently without locks or synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tryCreate(ref, accessor, playerRef, context) | CustomUIPage | O(1) | Factory method invoked by the Interaction System. Instantiates a new MemoriesPage using the player and interaction target from the provided context. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by game logic developers. It is a plugin for the server's interaction framework. A developer or designer would typically specify this supplier in a data file that configures an entity or block interaction. The system then resolves and invokes it automatically.

The following example illustrates how the *engine* would use this class, not a typical user.
```java
// This code is representative of how the server's OpenCustomUIInteraction handler
// would use the supplier. Developers do not call this directly.

// 1. The handler retrieves the configured supplier
OpenCustomUIInteraction.CustomPageSupplier supplier = new MemoriesPageSupplier();

// 2. The handler invokes the supplier with the current interaction context
CustomUIPage page = supplier.tryCreate(entityStoreRef, accessor, player, context);

// 3. The resulting page is sent to the player's UI manager
if (page != null) {
    player.getUIManager().openPage(page);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Invocation:** Do not manually create an instance of MemoriesPageSupplier and call tryCreate. The InteractionContext and other parameters are complex objects deeply tied to the server state and are provided exclusively by the calling Interaction System. Attempting to construct them manually will lead to unstable and unpredictable behavior.
- **Stateful Subclassing:** Do not extend this class to add state. The design contract of a supplier within the Interaction System assumes it is stateless and reusable. If you need a different page creation logic, create a new, independent implementation of the CustomPageSupplier interface.

## Data Pipeline
The MemoriesPageSupplier acts as a factory node in the data flow that begins with a player action and ends with a UI update on the client.

> Flow:
> Player Interaction Event -> Server Interaction System -> **MemoriesPageSupplier.tryCreate()** -> New MemoriesPage Instance -> Server UI Manager -> Network Message -> Client UI Render

