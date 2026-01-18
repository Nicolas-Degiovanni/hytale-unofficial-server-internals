---
description: Architectural reference for WeatherResetCommand
---

# WeatherResetCommand

**Package:** com.hypixel.hytale.builtin.weather.commands
**Type:** Transient

## Definition
```java
// Signature
public class WeatherResetCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The **WeatherResetCommand** class is an implementation of the Command Pattern, designed to integrate with the server's command processing system. It encapsulates the specific server action of resetting a world's weather back to its natural, dynamic cycle.

Architecturally, this class serves as a user-facing entry point that translates a simple text command, such as */weather reset*, into a direct state change within the game's weather simulation. It does not contain any weather logic itself; instead, it acts as a specialized delegate. Its primary function is to invoke the core weather modification logic, found in **WeatherSetCommand**, with a null parameter. This null value is the contractual signal to the weather system to clear any "forced" weather state, allowing the simulation to resume its normal progression.

By extending **AbstractWorldCommand**, it inherits the necessary boilerplate for registration, permission handling, and context injection, ensuring it operates safely within the server's execution model for world-modifying actions.

### Lifecycle & Ownership
- **Creation:** A single instance of **WeatherResetCommand** is created by the server's command registration system during the server bootstrap phase. The system discovers and instantiates all command classes to build its command map.
- **Scope:** The object instance is a stateless singleton that persists for the entire server lifetime. It holds no per-execution state; all necessary context (**World**, **CommandContext**) is passed into the **execute** method for each invocation.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server is shutting down or the module containing it is unloaded.

## Internal State & Concurrency
- **State:** This class is stateless and effectively immutable after its constructor runs. The only instance field is a static final **Message** object, which is a constant. All operations are performed on state passed in via method arguments.
- **Thread Safety:** The object itself is thread-safe due to its stateless nature. However, the **execute** method performs mutations on the **World** state. The server's command system guarantees safety by invoking all world commands sequentially on the main server thread (the "tick loop"). Direct, multi-threaded invocation of the **execute** method is unsupported and would lead to severe concurrency violations.

## API Surface
The public contract is defined by its constructor for framework instantiation and the overridden **execute** method for command invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Clears any forced weather state in the target **World**, reverting it to a natural cycle. Sends a confirmation message to the command issuer. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly via code. It is invoked by the server's command handler in response to a player or system typing the command into the console or chat.

*Example user-initiated command:*
```
/weather reset
```
or its alias:
```
/weather clear
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create an instance using `new WeatherResetCommand()`. The command system handles instantiation and registration. Manual creation results in a disconnected object that the server is unaware of.
- **Manual Invocation:** Avoid calling the **execute** method directly. Bypassing the command system skips critical infrastructure, including permission checks, argument parsing, and thread safety guarantees.

## Data Pipeline
The flow of data for this command begins with user input and ends with a world state modification that is read by the weather simulation engine.

> Flow:
> Player Chat Input (`/weather reset`) -> Network Layer -> Server Command Parser -> Command Dispatcher -> **WeatherResetCommand.execute()** -> WeatherSetCommand.setForcedWeather(world, null) -> World Storage Update -> Weather Simulation Engine reads cleared state

