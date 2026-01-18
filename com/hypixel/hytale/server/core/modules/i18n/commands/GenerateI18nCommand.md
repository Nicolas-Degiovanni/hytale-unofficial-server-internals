---
description: Architectural reference for GenerateI18nCommand
---

# GenerateI18nCommand

**Package:** com.hypixel.hytale.server.core.modules.i18n.commands
**Type:** Transient

## Definition
```java
// Signature
public class GenerateI18nCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts

The GenerateI18nCommand is a developer-focused, server-side administrative tool responsible for generating and updating default language files. It is not part of the runtime game loop but rather a utility for maintaining internationalization (i18n) keys.

Architecturally, this command acts as a **coordinator and aggregator**. Its primary design function is to decouple the various game systems that define translatable strings (e.g., items, blocks, UI elements) from the process that writes them to disk. It achieves this by leveraging the server's event bus.

When executed, the command does not possess knowledge of any specific translation keys. Instead, it dispatches a `GenerateDefaultLanguageEvent`. Other modules and systems across the server listen for this event and respond by populating a shared map with their required translation keys and default English values. Once all listeners have responded, this command proceeds to merge the aggregated data with existing on-disk language files and write the results.

This event-driven aggregation pattern is critical for modularity, ensuring that new game features can expose their translatable strings without modifying the core i18n generation logic.

## Lifecycle & Ownership

-   **Creation:** An instance of GenerateI18nCommand is created and registered by its parent module, the I18nModule, during server startup. It is managed by the central server Command System.
-   **Scope:** The command object is a long-lived singleton that persists for the entire server session. However, each execution of the command is a distinct, short-lived, and asynchronous operation.
-   **Destruction:** The object is destroyed and de-registered from the Command System when the server shuts down or the I18nModule is unloaded.

## Internal State & Concurrency

-   **State:** The command object itself is effectively stateless. Its only instance field, `cleanArg`, is an immutable definition of a command-line flag. All state related to a specific execution, such as the collection of translation maps, is created and scoped entirely within the `executeAsync` method call.

-   **Thread Safety:** This class is designed for concurrent operation and is thread-safe.
    -   The initial event dispatch occurs on the main server thread.
    -   A `ConcurrentHashMap` is used to safely collect translation data from event listeners, which may operate on different threads.
    -   All file I/O and merging logic is explicitly offloaded to a worker thread pool via `CompletableFuture.runAsync`. This prevents the file writing process from blocking the main server tick loop.

    **WARNING:** Direct modification of this class requires a deep understanding of Java's concurrency model to avoid introducing deadlocks or race conditions.

## API Surface

The public contract is defined by its role as a command. Direct invocation from other code is not a supported use case.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | O(N * M) | Triggers the event-driven translation key aggregation and file writing process. Complexity is dependent on N translation files and M keys per file. Fails if the base asset pack is immutable. |

## Integration Patterns

### Standard Usage

This command is not intended to be used programmatically. It is invoked through the server console or by an in-game administrator with appropriate permissions.

The primary integration point for developers is **not this class**, but the `GenerateDefaultLanguageEvent` it dispatches. To add new default translations to the generation process, a system must subscribe to this event.

```java
// In another module, e.g., an ItemRegistry
@Subscribe
public void onGenerateDefaultLanguage(GenerateDefaultLanguageEvent event) {
    TranslationMap itemTranslations = event.getOrCreateTranslationMap("items");
    for (Item item : allRegisteredItems) {
        itemTranslations.put(item.getTranslationKey(), item.getEnglishName());
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance with `new GenerateI18nCommand()`. The command will not be registered with the server and will be non-functional.
-   **Modifying for New Keys:** Do not modify this class to add hardcoded translation keys. This violates the core architectural principle of decoupling. Always use the event-based system described in Standard Usage.
-   **Blocking Execution:** Do not call `.get()` or `.join()` on the `CompletableFuture` returned by `executeAsync` from a synchronous context like the main server thread. This will freeze the server until all file I/O is complete.

## Data Pipeline

The flow of data for a single execution is a multi-stage, asynchronous process orchestrated by this command.

> Flow:
> Server Console Input (`/i18n gen`) -> Command System Routing -> **GenerateI18nCommand.executeAsync** -> Event Bus Dispatch (`GenerateDefaultLanguageEvent`) -> System Event Listeners -> Populate ConcurrentHashMap -> **GenerateI18nCommand (Async Task)** -> Merge with On-Disk Files -> Write to File System (`.lang` files)

