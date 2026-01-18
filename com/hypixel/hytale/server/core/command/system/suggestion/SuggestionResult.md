---
description: Architectural reference for SuggestionResult
---

# SuggestionResult

**Package:** com.hypixel.hytale.server.core.command.system.suggestion
**Type:** Transient

## Definition
```java
// Signature
public class SuggestionResult {
```

## Architecture & Concepts
The SuggestionResult class is a stateful builder object responsible for aggregating command completion suggestions. It serves as a temporary container within the server's command processing pipeline, decoupling the logic that *generates* suggestions from the final list sent to the client.

Its primary architectural role is to provide a unified interface for different suggestion sources. For example, a command argument parser for player names can use the fuzzy matching capabilities, while a parser for game modes can add simple, static suggestions to the same result object.

The inclusion of a `fuzzySuggest` method is a key design choice, indicating that the system is built to handle user error and provide a more intuitive, "did you mean?" style of command completion, rather than relying solely on prefix matching. This is critical for suggesting dynamic data like entity names or world identifiers where exact spelling is difficult.

## Lifecycle & Ownership
- **Creation:** A new SuggestionResult is instantiated on-demand by the command system whenever a client requests command completions. This typically occurs within the suggestion-providing logic of a command's argument parser (e.g., an `ArgumentType` implementation).
- **Scope:** The object's lifetime is extremely short and is strictly bound to the lifecycle of a single suggestion request. It is created, populated, and read from within the same logical transaction.
- **Destruction:** The instance becomes eligible for garbage collection immediately after its internal list is retrieved via `getSuggestions` and passed to the network serialization layer. There is no manual resource management or explicit destruction required.

## Internal State & Concurrency
- **State:** SuggestionResult is a **highly mutable** object. Its core state is an internal `ObjectArrayList` that accumulates string suggestions. All `suggest` methods directly mutate this internal list.
- **Thread Safety:** This class is **not thread-safe** and must be confined to a single thread. It is designed for synchronous, single-threaded use within the server's command processing logic. Accessing an instance from multiple threads will result in undefined behavior, likely a `ConcurrentModificationException` or a corrupted suggestion list.

**WARNING:** Do not share SuggestionResult instances across threads or asynchronous callbacks. The internal collection is not synchronized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| suggest(suggestion) | SuggestionResult | O(1) amortized | Adds a literal string to the list of suggestions. |
| getSuggestions() | List<String> | O(1) | Returns a reference to the internal list of collected suggestions. |
| fuzzySuggest(input, items, toStringFunction) | SuggestionResult | O(N * M) | Performs a fuzzy search across a collection of items. N is the number of items and M is the complexity of the fuzzy distance algorithm. Internally maintains a sorted list of the top 5 matches, making it more complex than a simple iteration. |

## Integration Patterns

### Standard Usage
SuggestionResult is intended to be used as a builder. A new instance is created, populated with suggestions from various sources, and then its final list is extracted for network transport.

```java
// Example from a hypothetical argument parser
public List<String> getSuggestionsFor(String partialInput) {
    SuggestionResult result = new SuggestionResult();
    Collection<Player> onlinePlayers = Server.getOnlinePlayers();

    // Use fuzzy logic for dynamic, error-prone data
    result.fuzzySuggest(partialInput, onlinePlayers, Player::getName);

    // Add a static, always-available suggestion
    if ("@all".startsWith(partialInput)) {
        result.suggest("@all");
    }

    return result.getSuggestions();
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not reuse a SuggestionResult instance for a new suggestion request. The internal list is never cleared, which would cause suggestions from the previous request to leak into the new one. Always create a new instance.
- **Deferred Population:** Do not pass a SuggestionResult to an asynchronous task to be populated later. The command system expects the suggestions to be available synchronously. This also violates thread-safety guarantees.

## Data Pipeline
SuggestionResult is a critical link in the data flow for command autocompletion.

> Flow:
> Client Input Packet -> Command Dispatcher -> Argument Parser -> **SuggestionResult (Creation & Population)** -> Command Dispatcher (Retrieval) -> Network Serializer -> Client Suggestion Packet

