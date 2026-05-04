# Firestore Authentication Tokens Guide

## Overview
When working with Firestore, authentication is typically handled via **Firebase Authentication**. Upon successful sign-in, Firebase issues an **ID Token** (a secure JSON Web Token or JWT) and a long-lived **Refresh Token**. 

- **ID Token:** A short-lived, securely signed JWT that identifies the user and evaluates database security rules.
- **Refresh Token:** A long-lived token used to seamlessly retrieve new ID tokens when they expire, without requiring the user to re-authenticate.

## Client-Side Best Practices
- **Let the Firebase SDK manage tokens:** The official Firebase client SDKs automatically manage token creation, storage, and refreshing. Avoid manually storing or refreshing tokens yourself unless absolutely necessary.
- **Never send raw UIDs to your backend:** If your app communicates with a custom backend or Cloud Functions, never send a raw `uid` in the request body (which can be easily spoofed). Instead, retrieve the ID token (e.g., using `auth.currentUser.getIdToken()`) and pass it in the `Authorization` HTTP header as a Bearer token.
- **Secure storage for manual handling:** If you must handle tokens manually (e.g., in React Native or native mobile apps), always store them in secure, encrypted storage like Android Keystore, iOS Keychain, or Expo SecureStore.

## Server-Side Verification
If you pass an ID token to your custom backend or Cloud Functions, it must be securely verified before processing API requests:
- **Always verify tokens with the Admin SDK:** Use the Firebase Admin SDK to verify the token signature and expiration: `admin.auth().verifyIdToken(token)`.
- **Trust the token, not the client:** Once the token is verified, the Admin SDK returns the decoded token payload. Always extract the user's `uid` and other data directly from this decoded payload. Ignore any `uid` the client might have provided in the request body.
- **Never log tokens:** ID tokens are sensitive credentials. Logging them in plain text can lead to account takeovers if your logs are accessed by unauthorized personnel or attackers.

## Firestore Security Rules & Custom Claims
- **Default to deny:** Never use `allow read, write: if true;` in production. Start by denying all access and opening up specific paths.
- **Use the `request.auth` object:** The `request.auth` variable is securely populated by the user's validated ID token. Always check that `request.auth != null` before granting access. Enforce data ownership by ensuring `request.auth.uid == resource.data.ownerId`.
- **Leverage Custom Claims for Role-Based Access Control (RBAC):** Instead of storing user roles (like "admin" or "editor") inside a Firestore document that requires an extra database read to verify, embed roles directly into the ID token using **Custom Claims**. You can set these via the Admin SDK and evaluate them synchronously in your rules (e.g., `allow write: if request.auth.token.admin == true;`). Keep custom claims small, as JWTs have a payload limit (usually up to 1000 bytes).

## Token Lifecycle & Security
- **Understand expiration times:** Firebase ID tokens expire exactly **one hour** after they are minted. The client SDK silently uses the refresh token to get a new ID token when it expires.
- **Handle Revocation on Logout:** ID tokens are stateless JWTs, meaning they remain valid until their 1-hour expiration time is up, even if the user logs out. To secure your application against hijacked sessions:
  - **Client-side:** Clear session storage/local storage and utilize `auth.signOut()` to drop the token from memory.
  - **Server-side:** If you suspect an account is compromised, use the Admin SDK to revoke all refresh tokens (`admin.auth().revokeRefreshTokens(uid)`).
- **Require recent sign-in for sensitive actions:** For destructive operations (like deleting an account or changing a password), use `auth_time` in the token payload to force the user to re-authenticate if their session is too old.
