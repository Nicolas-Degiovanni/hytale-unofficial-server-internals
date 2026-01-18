---
description: Architectural reference for MessageUtil
---

# MessageUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public class MessageUtil {
```

## Architecture & Concepts
MessageUtil is a core, stateless text-processing engine for the Hytale server. It acts as the primary bridge between abstract message representations and their final, human-readable string forms. The class serves two distinct architectural purposes: server-side console rendering and client-side message localization.

1.  **Console Rendering:** For server-side logging and console commands, MessageUtil translates the server's internal Message object graph into ANSI-formatted strings suitable for display in a terminal. This is handled by the toAnsiString method, which leverages the JLine library to convert color codes and message hierarchies into styled console output.

2.  **Internationalization (i18n) & Formatting:** The class contains a powerful, custom implementation of the ICU MessageFormat standard. The formatText method is the server's canonical way to perform localization. It parses templates for placeholders, substituting them with dynamic data and handling complex grammatical rules like pluralization. This component is critical for supporting multiple languages and ensuring consistent text formatting in messages sent to game clients. It integrates directly with the I18nModule to resolve message identifiers before formatting.

## Lifecycle & Ownership
-   **Creation:** As a static utility class, MessageUtil is never instantiated. The Java ClassLoader loads it into memory upon the first static access to any of its methods.
-   **Scope:** Application-level. Its methods are available throughout the entire lifecycle of the server JVM.
-   **Destruction:** The class and its static methods are unloaded from memory only when the JVM process terminates. There is no instance-level state to manage or clean up.

## Internal State & Concurrency
-   **State:** **Stateless and Immutable**. MessageUtil contains no member fields and maintains no state between method invocations. All computations are performed exclusively on the arguments provided to each static method call.
-   **Thread Safety:** **Fully thread-safe**. Due to its stateless nature, all public methods can be safely and concurrently invoked from any thread without external synchronization. This is critical, as message formatting may be triggered by network threads, game loop ticks, or asynchronous background tasks.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toAnsiString(Message) | AttributedString | O(N) | Recursively traverses a Message object graph and converts it into a JLine AttributedString for colored console output. N is the total number of nodes in the message tree. |
| formatText(String, Map, Map) | String | O(L) | Parses and formats a string template according to ICU MessageFormat-like rules. Substitutes parameters and handles pluralization. L is the length of the template string. Complexity can increase with deeply nested message lookups. |
| sendSuccessReply(PlayerRef, int, Message) | void | O(1) | **Deprecated.** Constructs and sends a SuccessReply packet. Direct network access from a utility is discouraged. |
| sendFailureReply(PlayerRef, int, Message) | void | O(1) | **Deprecated.** Constructs and sends a FailureReply packet. Direct network access from a utility is discouraged. |
| hexToStyle(String) | AttributedStyle | O(1) | A low-level helper that converts a hex color string into a JLine style object. |

## Integration Patterns

### Standard Usage
MessageUtil is most commonly used in conjunction with the I18nModule to prepare localized text that will be sent to a player. The caller first retrieves a raw, localized template and then uses MessageUtil to inject dynamic data.

```java
// In a module that needs to send a localized message to a player
Map<String, ParamValue> params = new HashMap<>();
params.put("playerName", new StringParamValue(player.getName()));
params.put("zoneName", new StringParamValue("Emerald Grove"));

// 1. Retrieve the raw, localized template from the i18n system
String template = I18nModule.get().getMessage("en-US", "world.enter_zone");
// Template might be: "Player {playerName} has entered {zoneName}."

// 2. Use MessageUtil to format the final string
String formattedMessage = MessageUtil.formatText(template, params, null);

// 3. Use the result in a packet or other system
player.sendMessage(formattedMessage);
```

### Anti-Patterns (Do NOT do this)
-   **Using Deprecated Network Methods:** Do not call the sendSuccessReply or sendFailureReply methods. This pattern violates separation of concerns by mixing a presentation-layer utility with network-layer logic. Packet creation and dispatch should be handled exclusively by a player's PacketHandler or a dedicated network service.
-   **Complex Logic in Templates:** Avoid embedding application logic within the message templates. The formatting syntax is designed for localization and presentation, not for general-purpose programming. Complex conditional text should be resolved in Java code before calling formatText.
-   **Manual String Concatenation:** Do not build localized strings using manual concatenation (e.g., "Welcome, " + player.getName()). This practice prevents proper localization and is susceptible to grammatical errors in other languages. Always use formatText with parameterized templates.

## Data Pipeline
MessageUtil sits at the end of the text localization pipeline, transforming structured data into a final string representation for a specific output channel.

**For Client-Bound Messages:**
> Flow:
> I18n Key + Parameters -> I18nModule -> Raw Localized Template -> **MessageUtil.formatText** -> Final Formatted String -> Network Packet -> Client UI

**For Server Console Output:**
> Flow:
> Internal Game Event -> Message Object -> **MessageUtil.toAnsiString** -> JLine AttributedString -> Console Rendering Engine

