---
description: Architectural reference for IAuthCredentialStore
---

# IAuthCredentialStore

**Package:** com.hypixel.hytale.server.core.auth
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IAuthCredentialStore {
    // Nested Record
    public record OAuthTokens(@Nullable String accessToken, @Nullable String refreshToken, @Nullable Instant accessTokenExpiresAt) { ... }
}
```

## Architecture & Concepts
The IAuthCredentialStore interface defines a strict contract for the secure, persistent storage of client-side authentication credentials. It serves as the single source of truth for the current user's session identity, including OAuth 2.0 tokens and the associated profile UUID.

This component acts as a high-level abstraction over the physical storage medium, which could be an encrypted file, the operating system's keychain, or another secure enclave. By defining this interface, the core authentication logic is completely decoupled from the underlying storage implementation, allowing for platform-specific strategies without altering the authentication flow. It is a critical dependency for any system that needs to make authenticated requests to backend services, such as the API client or the game server connector.

## Lifecycle & Ownership
As an interface, IAuthCredentialStore itself has no lifecycle. However, any concrete implementation is subject to the following expectations:

-   **Creation:** A concrete implementation (e.g., FileBasedAuthCredentialStore) is expected to be instantiated once at the beginning of the client application's lifecycle, typically by a central service registry or dependency injection container.
-   **Scope:** The implementation must be a **singleton** that persists for the entire duration of the application session. Its state represents the currently logged-in user.
-   **Destruction:** The object is destroyed upon application shutdown. The `clear` method dictates the logical destruction of a user's session data (e.g., on logout), but the object instance itself remains active.

## Internal State & Concurrency
-   **State:** The contract is inherently stateful and mutable. It is designed to hold and manage the OAuthTokens and profile UUID, which change as the user logs in, refreshes their session, or logs out. Implementations are responsible for persisting this state across application restarts.
-   **Thread Safety:** **CRITICAL:** All implementations of this interface **must be thread-safe**. The credential store will be accessed by multiple threads concurrently. For example, the main game thread might check for a valid session, while a background network thread attempts to refresh an expired access token. Implementations must use appropriate synchronization mechanisms (e.g., synchronized methods, locks) to prevent race conditions and data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setTokens(OAuthTokens tokens) | void | O(1) | Atomically overwrites the stored OAuth tokens. This operation must persist the data to the underlying storage medium. |
| getTokens() | OAuthTokens | O(1) | Retrieves the current OAuth tokens from storage. Returns a non-null record, which may contain null token values if not set. |
| setProfile(UUID profileId) | void | O(1) | Associates a user profile UUID with the stored credentials. Persists the change. |
| getProfile() | UUID | O(1) | Retrieves the associated user profile UUID. Returns null if no user is authenticated. |
| clear() | void | O(1) | Purges all stored credentials and the profile UUID, effectively logging the user out. This is the primary mechanism for invalidating a session. |

## Integration Patterns

### Standard Usage
Systems should never be aware of the concrete implementation. Always request the IAuthCredentialStore from the central service context and program against the interface.

```java
// Correctly retrieve the store from a service registry
IAuthCredentialStore credentialStore = serviceContext.getService(IAuthCredentialStore.class);

// Use the store to prepare an authenticated API call
IAuthCredentialStore.OAuthTokens tokens = credentialStore.getTokens();
if (tokens.isValid()) {
    apiClient.setAuthHeader(tokens.accessToken());
    apiClient.makeRequest();
}
```

### Anti-Patterns (Do NOT do this)
-   **Local Caching:** Do not cache the results of `getTokens` in local variables for extended periods. The tokens can be updated in the background by a refresh thread. Always fetch the latest tokens from the store before making an authenticated request.
-   **Implementation Casting:** Never cast the interface to a concrete type to access implementation-specific methods. This breaks the abstraction and couples your code to a specific storage strategy.
-   **Neglecting `clear`:** Failure to call `clear` on user logout will leave sensitive credentials on disk. This is a critical security risk and will cause the next application launch to incorrectly assume the user is still logged in.

## Data Pipeline
The credential store is a terminal point for authentication data from the login flow and a starting point for data in authenticated API requests.

> Flow:
> User Input (Login Form) -> Authentication Service -> **IAuthCredentialStore (Implementation)** -> API Client -> Backend Service Request (with Authorization Header)

