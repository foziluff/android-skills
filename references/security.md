# Security

> Paths and code below come from the reference project DebtNote (`com.wigroup.*`) as evidence; the rules apply to any
> app built with this skill. *(if the app …)* / *(optional)* marks capabilities only some apps have. A verified universal
> version of the base code is in [templates/project](../templates/project) and [templates/feature](../templates/feature).

Assets of the app: the user's private financial notes (names, amounts, comments), the auth token, the Pro status.
Threats: another app on the device, someone holding the unlocked phone, a lost/backed-up device, a network attacker,
logcat/bug-report leaks, a tampered server response.

Markers as in SKILL.md. Most rules are **[hypothesis]** — inferred from the code; the author has not confirmed the threat model.

---

## 1. Token storage

`data/.../platform/crypto/CryptoManager.kt` + `data/.../platform/TokenGatewayImpl.kt`
```kotlin
@Singleton
internal class CryptoManager @Inject constructor() {
    private val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }

    private fun createSecretKey(): SecretKey {
        val keyGenerator = KeyGenerator.getInstance(ALGORITHM, "AndroidKeyStore")
        keyGenerator.init(
            KeyGenParameterSpec.Builder(ALIAS, KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
                .setBlockModes(BLOCK_MODE)                  // GCM
                .setEncryptionPaddings(PADDING)             // NONE
                .setRandomizedEncryptionRequired(true)
                .build()
        )
        return keyGenerator.generateKey()
    }

    fun encrypt(data: String): ByteArray {
        val cipher = Cipher.getInstance(TRANSFORMATION)
        cipher.init(Cipher.ENCRYPT_MODE, getSecretKey())
        return cipher.iv + cipher.doFinal(data.toByteArray(Charsets.UTF_8))   // IV (12 bytes) prepended
    }

    fun decrypt(encryptedDataWithIv: ByteArray): String? = runCatching { … GCMParameterSpec(TAG_LENGTH, iv) … }.getOrNull()
}

override suspend fun saveToken(token: String) {
    val encryptedToken = cryptoManager.encrypt(token)
    dataStore.edit { it[tokenKey] = encryptedToken }            // byteArrayPreferencesKey("access_token")
}
```
- The key never leaves AndroidKeyStore; AES/GCM (key size = Keystore default, not set explicitly) with a random IV per
  encryption; ciphertext + IV in DataStore.
- If decryption fails (key lost after restore, tampering) → `null` → the user is simply logged out. Fail closed.
- *Why not EncryptedSharedPreferences:* [hypothesis] it is deprecated and pulls Tink; Keystore + DataStore is platform-only.
- Rule: any new secret (refresh token, PIN) uses `CryptoManager` with its own alias; never plain DataStore.
- legacy: `decrypt` calls `printStackTrace()` on failure — drop it.

## 2. Authentication flows

- Password login: input validated in domain (`LoginInputValidator`: lengths + allowed characters) before the request.
- Google login via Credential Manager with a fresh `SecureRandom` 32-byte nonce per request (`generateSecureRandomNonce`);
  only the ID token goes to the backend (`auth/google`), which must verify it.
- The client id is not hard-coded in the analysed copy (`serverClientId = ""`) **[question to the author: where is it
  injected?]** — it must come from build config, not from source.
- After login: save token → fetch profile; if the profile fails, the token is cleared (`LoginUseCase`) — no half-logged-in state.
- `401` anywhere in sync → global logout (`SyncType.LOGOUT` → `LogoutUseCase`), local data wiped in one transaction,
  Firebase installation id deleted.
- Device management (`devices` endpoints) lets the user revoke other sessions.

## 3. App lock

- `BlockingAppType` (`NONE / IMMEDIATELY / AFTER_1_MINUTE / AFTER_30_MINUTE`); `MainViewModel.onStart` compares
  `now - lastBackgroundTime` and shows `AuthenticateScreen` on top of the NavGraph (back handling disabled while locked).
- `BiometricPrompt` with `BIOMETRIC_STRONG or BIOMETRIC_WEAK or DEVICE_CREDENTIAL`.
- **Gap:** `authenticate()` calls `onAuthenticationSucceeded()` when `canAuthenticate` is false (no screen lock / no
  biometrics enrolled) — the lock fails **open**. **[question to the author]** If intentional (user removed the device lock),
  document it in the setting's UI; otherwise disable the option when the device cannot authenticate.
- `isAppLocked` lives in ViewModel memory: after process death the lock is re-evaluated in `onStart` from the persisted
  `lastBackgroundTime` — correct, keep it that way.

## 4. Input and SQL

- Every user string is trimmed and length-checked in domain; the UI also caps input (`maxLength`).
- SQL: static `@Query` with `:params`; dynamic `RoomRawQuery` with `bindText`; `ORDER BY`/`HAVING` from enums only.
- Amounts: filtered by `currencyFilterValue` (digits, one dot, two decimals, ≤ `DebtInputValidator.MAX_VALUE`) and validated
  again in `DebtInputValidator`.

## 5. Data exposure

- No logging at all (no `Log`, Timber or Crashlytics). Keep it: names, amounts, emails, tokens must never be printed.
- Analytics: Firebase Analytics is on the classpath **[question: which events are sent?]** — never put PII in event params.
- Notifications show only server-provided localization keys mapped to local strings (`pro_title`, `pro_body`) or plain
  notification text; the big picture is downloaded only from the push payload URL.
- PDF backup: written to `cacheDir/backups/`, shared through `FileProvider` (`file_paths.xml` exposes only `cache-path
  backups/`) with `FLAG_GRANT_READ_URI_PERMISSION`. **[proposal]** delete old backup files after sharing.

## 6. Manifest and components

The analysed copy of the reference project has **no `app/src/main/AndroidManifest.xml`**, so its declarations could not be
verified. The skeleton manifest ([templates/project/app/AndroidManifest.xml.stub](../templates/project/app/AndroidManifest.xml.stub))
follows these rules: only the launcher Activity is exported, `networkSecurityConfig` forbids cleartext, backup rules exclude
the database and DataStore.
Rules for any manifest change:
- `android:exported="false"` for every component except the launcher Activity; FCM service `exported="false"` with the
  Firebase intent filter.
- FileProvider `exported="false"`, `grantUriPermissions="true"`, authority `${applicationId}.fileprovider`.
- Only permissions that are used: `INTERNET`, `POST_NOTIFICATIONS`, `RECORD_AUDIO` (voice input, requested at use),
  `FOREGROUND_SERVICE` + `FOREGROUND_SERVICE_DATA_SYNC` (sync worker), `READ_CONTACTS` only if the picker needs it.
- `PendingIntent` always `FLAG_IMMUTABLE` (both places do).

## 7. Configuration
<a id="7-configuration"></a>

- `BASE_URL` is a `const val` in `data/remote/ApiFactory.kt` (HTTPS). **[proposal]** move to `buildConfigField` per build
  type (debug → staging, release → production) and never commit staging credentials.
- Never put production URLs, keystores, signing passwords, API keys in skill files, issues or commits.
- `local.properties` is git-ignored; signing configs are not in the build files (signing happens outside the repo
  **[question]**).

## 8. Network

- HTTPS only; Bearer token in a header (never in a URL/query).
- No certificate pinning, no `network_security_config` visible. **[proposal]** `network_security_config.xml` with
  `cleartextTrafficPermitted="false"`; pinning only together with a rotation plan (two pins).
- `ignoreUnknownKeys = true` is safe; do **not** enable `isLenient` or `coerceInputValues` without a reason.
- Pro status is computed against **server time** (`meta.server_time`) — device clock changes do not unlock Pro. Offline
  fallback re-evaluates `isPro` with the device clock (`UserRepositoryImpl.sync().onError`) **[question: accepted risk?]**.

## 9. Backups

- `backup_rules.xml` / `data_extraction_rules.xml` are still the Android Studio samples (no rules).
  With auto-backup enabled, Room (`debt_note.db`) and DataStore (`settings.preferences_pb`) go to the user's cloud backup;
  the encrypted token is useless after restore (Keystore key is not backed up) → user is logged out, local data restored.
  **[question to the author: is cloud backup of debts desired?]** If not: exclude `database` and `sharedpref`/`file` domains,
  or `allowBackup="false"`.

## 10. Release build

- `buildTypes.release.optimization.enable = true` (R8 shrinking/obfuscation). Keep rules are empty — kotlinx.serialization,
  Retrofit and Room ship their own consumer rules. Add rules only for reflection you introduce.
- `debuggable` is never set manually; no debug menus/backdoors in code.

## 11. Known gaps (fix when touching the code)

| Gap | Where | Target |
|---|---|---|
| App lock fails open without device credential | `util/Utils.authenticate` | disable option / explain |
| `printStackTrace` | 4 places | remove |
| Unstructured scope + new OkHttp client per image | `MyFirebaseMessagingService` | worker or owned scope, injected client |
| Backup rules are samples | `res/xml/*rules.xml` | explicit include/exclude |
| Base URL in code | `ApiFactory` | `buildConfigField` |
| Old PDF backups stay in cache | `PdfGenerator` | cleanup |

---

## Security checklist (every change)

- [ ] No `Log`, `printStackTrace`, `println`; no PII/tokens in analytics, notifications, exceptions or file names.
- [ ] Secrets only through `TokenGateway`/`CryptoManager`; nothing secret in code, resources, VCS or the skill.
- [ ] User input validated in domain (length, charset, range) and capped in UI.
- [ ] SQL parameters bound; no string-concatenated user input.
- [ ] New manifest components `exported="false"` unless required; new permissions justified and requested at use.
- [ ] `PendingIntent` with `FLAG_IMMUTABLE`; FileProvider paths minimal.
- [ ] New network calls via `safeApiCall`, HTTPS, token only in the header.
- [ ] Release build passes R8; no debug-only code paths in release.
