---
description: Architectural reference for ServerAuthManager
---

# ServerAuthManager

**Package:** com.hypixel.hytale.server.core.auth
**Type:** Singleton

## Definition
```java
// Signature
public class ServerAuthManager {
```

## Architecture & Concepts

The ServerAuthManager is the central authority for a Hytale server's identity and authentication lifecycle. It acts as a state machine and service client, managing the server's own credentials and its session with Hytale's backend services. This component is critical for both online multiplayer servers, which must authenticate to list themselves and validate players, and for singleplayer instances to establish an owner identity.

Its core responsibilities include:

*   **Multi-modal Authentication:** Implements several strategies for acquiring authentication tokens, including command-line arguments, environment variables, and interactive OAuth 2.0 flows (Device and Browser). This flexibility supports headless server deployments and local user-driven setups.
*   **Session Management:** Manages the entire lifecycle of a `GameSessionResponse`. This includes initial creation, periodic background refreshing to prevent expiration, and graceful termination during server shutdown.
*   **Credential Persistence:** Integrates with an extensible `IAuthCredentialStore` system, allowing OAuth tokens to be securely stored and reused across server restarts, enabling unattended session restoration.
*   **State Segregation:** Encapsulates all authentication-related state, such as tokens, profile information, and certificate data. It provides a clean, thread-safe interface for other parts of the server engine to consume this state without needing to understand the underlying authentication mechanisms.
*   **Asynchronous Operations:** Leverages `CompletableFuture` for long-running OAuth flows and a dedicated `ScheduledExecutorService` for background token refreshes. This ensures that authentication processes do not block the main server thread.

The ServerAuthManager is the bridge between the server's runtime environment and the external Hytale session service, making it a foundational component for any server operating in an online capacity.

### Lifecycle & Ownership

-   **Creation:** The single instance is created lazily via a classic double-checked locking pattern within the static `getInstance` method. However, the instance is inert until its `initialize` and `initializeCredentialStore` methods are explicitly called by the server's main bootstrap sequence, likely within `HytaleServer`.
-   **Scope:** As a global singleton, the ServerAuthManager persists for the entire lifetime of the server JVM process. Its state represents the server's identity for the duration of its execution.
-   **Destruction:** The manager is not garbage collected. A `shutdown` method is provided for graceful cleanup. This method cancels pending background tasks, terminates the active game session with the backend service, and shuts down its internal thread pool. This is a critical step to prevent orphaned sessions.

## Internal State & Concurrency

-   **State:** The ServerAuthManager is highly stateful and mutable. Its core state is composed of `volatile` fields and objects wrapped in atomic containers to manage concurrency. Key state includes:
    -   `gameSession`: An `AtomicReference` holding the current session tokens.
    -   `credentialStore`: An `AtomicReference` to the persistent token storage mechanism.
    -   `authMode`: A `volatile` enum tracking the current authentication method.
    -   `tokenExpiry`: A `volatile` `Instant` tracking when the current tokens expire.
    -   `pendingProfiles`: A `volatile` array holding user profiles during an interactive login that requires user selection.
    -   `refreshScheduler`: A single-thread `ScheduledExecutorService` dedicated to background token refresh tasks.

-   **Thread Safety:** This class is designed to be thread-safe and is accessed from multiple contexts, including the main server thread, command handlers, and its own background refresh thread.
    -   The singleton instance is protected by a `synchronized` block.
    -   Mutable shared state is managed through `AtomicReference`, `ConcurrentHashMap`, and `volatile` keywords to ensure visibility and atomicity.
    -   All background refresh logic is confined to the single-threaded `refreshScheduler`, preventing race conditions within the refresh cycle itself.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInstance() | ServerAuthManager | O(1) | Accesses the global singleton instance. |
| initialize() | void | O(1) | Initializes the manager by parsing CLI options and environment variables. Must be called once at startup. |
| initializeCredentialStore() | void | O(1) + I/O | Attempts to load credentials from the configured store and restore a previous session. |
| startFlowAsync(flow) | CompletableFuture | O(1) | Initiates an asynchronous OAuth flow. Returns a future that completes with the authentication result. |
| selectPendingProfile(index) | boolean | O(1) + Network | Completes an authentication flow when multiple game profiles are available for the authenticated account. |
| getIdentityToken() | String | O(1) | Retrieves the current identity token, if one exists. Returns null if unauthenticated. |
| getServerCertificateFingerprint() | String | O(1) | Computes and returns the SHA-256 fingerprint of the server's X.509 certificate. |
| logout() | void | O(1) | Clears all credentials, terminates the session, and resets the manager to an unauthenticated state. |
| shutdown() | void | O(1) + Network | Performs graceful shutdown by terminating the remote session and stopping background threads. |

## Integration Patterns

### Standard Usage

The ServerAuthManager is a low-level engine component. Other systems typically retrieve the instance to access the server's identity or authentication status.

```java
// Retrieve the singleton instance
ServerAuthManager authManager = ServerAuthManager.getInstance();

// Check the current authentication status for display or logic branching
String status = authManager.getAuthStatus();
System.out.println("Server auth status: " + status);

// Retrieve the identity token to pass to other services
String identityToken = authManager.getIdentityToken();
if (identityToken != null) {
    // Use token for backend API calls
}
```

### Anti-Patterns (Do NOT do this)

-   **Calling Methods Before Initialization:** Calling methods like `getGameSession` before `initialize` has been successfully executed will result in null pointers or undefined behavior. The manager relies on the server's option parsing to be complete.
-   **Ignoring Asynchronous Results:** The `startFlowAsync` methods return a `CompletableFuture`. Failing to handle the completion of this future means the authentication result will be lost, and the server will not become authenticated even if the user completes the flow.
-   **Storing Tokens Manually:** Do not extract tokens from the manager and attempt to store them yourself. Use the provided `IAuthCredentialStore` mechanism by configuring the correct `AuthCredentialStoreProvider` to ensure tokens are handled securely and correctly.
-   **Unsafe State Observation:** When accessing the `GameSessionResponse` object, be aware that it can be replaced at any time by the background refresh thread. Always retrieve the latest reference from the `AtomicReference` via `getGameSession` before use.

## Data Pipeline

The ServerAuthManager orchestrates complex data flows for authentication and session management.

**Flow 1: Initial Authentication via OAuth Device Flow**

> Command (`/auth login`) -> `startFlowAsync(deviceFlow)` -> **ServerAuthManager** -> `OAuthClient` -> HTTP Request to Auth Service -> User Code & URL returned -> User authenticates in browser -> **ServerAuthManager** (via `CompletableFuture`) -> `createGameSessionFromOAuth` -> `SessionServiceClient` -> HTTP Request for Profiles -> Profile list returned -> User selects profile (`/auth select`) -> `completeAuthWithProfile` -> `SessionServiceClient` -> HTTP Request to create session -> Game Session Tokens returned -> **ServerAuthManager** state updated -> `scheduleRefresh` is initiated.

**Flow 2: Automated Token Refresh**

> `ScheduledExecutorService` timer fires -> `doRefresh` method invoked on refresh thread -> **ServerAuthManager** -> `refreshGameSession` -> `SessionServiceClient` -> HTTP Request to refresh session -> New Game Session Tokens returned -> **ServerAuthManager** atomically updates `gameSession` reference -> New refresh is scheduled based on new token's expiry.

