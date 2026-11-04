# Mobile App Security Requirements

These rules are mandatory for all mobile app code generation. They extend the root CLAUDE.md and work alongside Project CodeGuard's mobile/platform security rules.

## Secure Storage

- **iOS**: Use Keychain for all sensitive data. Never UserDefaults, plist, or plain files.
- **Android**: Use Android Keystore (directly, or via DataStore with Tink using a Keystore-backed key). Never SharedPreferences or SQLite for secrets. Do not adopt `EncryptedSharedPreferences` / Jetpack Security Crypto in new code: Google deprecated the whole library in version 1.1.0 and no further releases are planned; migrate existing uses.
- **React Native**: Use react-native-keychain, not AsyncStorage for secrets.
- **Flutter**: Use flutter_secure_storage, not shared_preferences for secrets.

For implementation patterns, see: `/docs/security-patterns/secrets-management.md`

## Network Security

- All API calls must use HTTPS. No HTTP connections.
- Implement certificate pinning for company API connections.
- Never disable App Transport Security (iOS) in production.
- Never set android:usesCleartextTraffic="true" in production.
- Never bypass SSL certificate validation.

## Data Protection

- Never log sensitive data (credentials, PII, tokens, financial data).
- Implement screenshot prevention for sensitive screens.
- Clear sensitive data from memory when app backgrounds.
- Implement session timeouts requiring re-authentication.
- Mask sensitive data in UI (account numbers, SSN, etc.).

## Authentication and Session Management

- Use OAuth 2.0 with PKCE for mobile auth flows.
- Store tokens in secure storage only (Keychain/Keystore).
- Implement biometric auth (Face ID, Touch ID, fingerprint) where appropriate.
- Validate deep links to prevent unauthorized access.
- Set secure cookie attributes: HttpOnly, Secure, SameSite=Strict.
- Implement token rotation on refresh. Invalidate old tokens immediately.
- Enforce session expiry and re-authentication for sensitive operations.

## Build Security

- Enable code obfuscation (ProGuard/R8) for Android release builds.
- Remove all debug code and test credentials before release.
- Remove hardcoded test API endpoints.
- Do not include internal API documentation or developer comments in release builds.
