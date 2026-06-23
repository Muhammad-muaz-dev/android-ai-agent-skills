---
name: security
description: Secure Android apps with threat modeling, encrypted storage, network security, certificate pinning, auth hardening, obfuscation, tamper resistance, and privacy controls.
---

Scope
This skill governs every security surface in an Android app — storage, network, authentication, encryption, input validation, WebView, obfuscation, privacy, and third-party dependency risk. It begins when a security concern arises and ends when the implementation is threat-resistant, privacy-compliant, and maintainable.

It does not own UI design, business logic, navigation architecture, or unrelated feature development.

Core Security Principles (always apply)
Principle	Meaning
Least Privilege	Request only the permissions and access actually needed.
Defense in Depth	Layer multiple controls; never rely on a single security mechanism.
Assume Breach	Design as if the device, network, or process could be compromised.
Zero Trust Input	Every external input — user, network, IPC, deep link — is untrusted until validated.
Fail Secure	On error, default to the most restrictive/safe state. Never silently open access.
Security ≠ Obscurity	Obfuscation supplements; it does not replace real security controls.
Quick Decision Tree
Security concern?
│
├─ Sensitive data on-device          → Secure storage (§A)
├─ Network communication             → Network security + TLS (§B)
├─ SSL pinning needed                → Certificate / public key pinning (§B-3)
├─ Authentication tokens / sessions  → Auth security (§C)
├─ API keys / secrets                → Secret management (§D)
├─ Encrypting data                   → Encryption (§E)
├─ User / network / file input       → Input validation (§F)
├─ WebView in the app                → WebView hardening (§G)
├─ Code obfuscation                  → ProGuard / R8 (§H)
├─ Play Integrity / tamper detection → Integrity checks (§I)
├─ Logging in prod                   → Secure logging (§J)
├─ Third-party libraries             → Dependency audit (§K)
├─ Privacy & data minimization       → Privacy controls (§L)
│
└─ Full security review requested    → Security audit (§M)

Mandatory Rules
#	Rule
R1	Never hardcode API keys, secrets, passwords, or tokens anywhere in source code or resources.
R2	Never disable certificate validation (TrustManager that accepts all, HostnameVerifier that returns true).
R3	Never log passwords, tokens, keys, PII, or stack traces in production builds.
R4	Never store passwords in plain text, SharedPreferences (unencrypted), or external storage.
R5	Never implement custom cryptography. Use platform-provided or well-audited libraries (JCA, Tink, Security library).
R6	Never use deprecated algorithms: MD5, SHA-1 (for security), DES, 3DES, ECB mode, RC4.
R7	Never expose sensitive data in URLs, query parameters, or logs — use request bodies with HTTPS.
R8	All network communication must use HTTPS only. HTTP must be blocked in network_security_config.xml.
R9	All cryptographic keys must be stored in the Android Keystore — never in SharedPreferences, files, or constants.
R10	Every external input (user, network, IPC, deep link, file) must be validated and sanitized before use.
Implementation Checklist
Secure storage
 Tokens and sensitive prefs stored via EncryptedSharedPreferences
 Sensitive files stored via EncryptedFile
 Cryptographic keys in Android Keystore
 No passwords stored locally in any form
 Sensitive data cleared on logout
Network security
 network_security_config.xml blocks all cleartext traffic
 Certificate or public-key pinning implemented for sensitive endpoints
 Auth tokens sent in Authorization header, never in URL
 Server responses validated (status + schema) before use
Authentication
 Tokens stored in EncryptedSharedPreferences, never in plain prefs
 Token refresh implemented without exposing credentials
 Session fully invalidated on logout (token deleted + server-side revocation)
 Biometric authentication uses BiometricPrompt with Keystore-backed key
Encryption
 AES-256-GCM for symmetric encryption
 Keystore-generated key, never hardcoded
 IV never reused; included alongside ciphertext
 Tink used for envelope encryption where appropriate
Input validation
 All user input validated (length, format, type) before use
 All deep-link / Intent extras validated before processing
 File paths canonicalized and jail-checked before access
 Network responses schema-validated (Zod/Moshi/Gson strict mode)
Obfuscation & build
 R8 full-mode enabled in release builds
 Debug logging stripped via Timber or BuildConfig.DEBUG gates
 debuggable false in release buildType
 allowBackup set per policy (usually false for sensitive apps)
§A — Secure Storage
§A-1 — EncryptedSharedPreferences (tokens, settings)
// security/storage/SecurePreferences.kt
class SecurePreferences(context: Context) {
    private val prefs: SharedPreferences by lazy {
        val masterKey = MasterKey.Builder(context)
            .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
            .build()
        EncryptedSharedPreferences.create(
            context,
            "secure_prefs",
            masterKey,
            EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
            EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
        )
    }
    fun putString(key: String, value: String) = prefs.edit().putString(key, value).apply()
    fun getString(key: String): String?        = prefs.getString(key, null)
    fun remove(key: String)                    = prefs.edit().remove(key).apply()
    /** Call on logout — wipe all sensitive data. */
    fun clearAll()                             = prefs.edit().clear().apply()
}
// Dependency: implementation("androidx.security:security-crypto:1.1.0-alpha06")

§A-2 — EncryptedFile (sensitive file storage)
// security/storage/SecureFileStorage.kt
object SecureFileStorage {
    fun write(context: Context, filename: String, data: ByteArray) {
        val file = File(context.filesDir, filename)
        val masterKey = MasterKey.Builder(context)
            .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
            .build()
        EncryptedFile.Builder(
            context, file, masterKey,
            EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB
        ).build().openFileOutput().use { it.write(data) }
    }
    fun read(context: Context, filename: String): ByteArray {
        val file = File(context.filesDir, filename)
        if (!file.exists()) return ByteArray(0)
        val masterKey = MasterKey.Builder(context)
            .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
            .build()
        return EncryptedFile.Builder(
            context, file, masterKey,
            EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB
        ).build().openFileInput().use { it.readBytes() }
    }
    fun delete(context: Context, filename: String) {
        File(context.filesDir, filename).delete()
    }
}

§A-3 — What NOT to use
// NEVER — plain SharedPreferences for tokens
getSharedPreferences("prefs", MODE_PRIVATE)
    .edit().putString("token", authToken).apply()    // ← plaintext, readable on rooted devices
// NEVER — external storage for sensitive files
File(Environment.getExternalStorageDirectory(), "token.txt").writeText(token)  // ← world-readable
// NEVER — hardcoded in constants
companion object { const val API_SECRET = "abc123xyz" }  // ← extracted from APK trivially

§B — Network Security
§B-1 — Network Security Configuration
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <!-- Block ALL cleartext traffic app-wide -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
            <!-- Do NOT include user certificates in production -->
        </trust-anchors>
    </base-config>
    <!-- Optional: allow cleartext to localhost for debug only -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" />   <!-- debug only — never in release -->
        </trust-anchors>
    </debug-overrides>
</network-security-config>

<!-- AndroidManifest.xml -->
<application
    android:networkSecurityConfig="@xml/network_security_config"
    android:usesCleartextTraffic="false"
    android:debuggable="false"     <!-- always false in release -->
    android:allowBackup="false"    <!-- disable adb backup for sensitive apps -->
    ...>

§B-2 — Secure OkHttp Client
// security/network/SecureHttpClient.kt
object SecureHttpClient {
    fun build(): OkHttpClient = OkHttpClient.Builder()
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .addInterceptor(AuthInterceptor())        // attach token
        .addInterceptor(SanitizedLoggingInterceptor()) // safe logging (§J)
        .apply {
            if (BuildConfig.DEBUG) {
                // Never add Charles/Proxy certs in release
                addNetworkInterceptor(HttpLoggingInterceptor().apply {
                    level = HttpLoggingInterceptor.Level.HEADERS // never BODY in prod
                })
            }
        }
        .build()
}
class AuthInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val token = SessionManager.getToken() // from EncryptedSharedPreferences
        val request = if (token != null) {
            chain.request().newBuilder()
                .header("Authorization", "Bearer $token")
                // NEVER put token in URL query params
                .build()
        } else {
            chain.request()
        }
        return chain.proceed(request)
    }
}

§B-3 — SSL / Certificate Pinning
// Prefer public key pinning over certificate pinning — keys survive cert renewal.
// Generate pin: openssl s_client -connect api.example.com:443 | openssl x509 -pubkey -noout |
//               openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64
object PinningConfig {
    // Minimum two pins (primary + backup) — never deploy with only one.
    private const val PRIMARY_PIN = "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA="
    private const val BACKUP_PIN  = "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB="
    fun build(): CertificatePinner = CertificatePinner.Builder()
        .add("api.example.com", PRIMARY_PIN, BACKUP_PIN)
        .build()
}
// Attach to OkHttpClient
OkHttpClient.Builder()
    .certificatePinner(PinningConfig.build())
    .build()

// §B-3b — Pin rotation strategy
// 1. Deploy new cert to server.
// 2. Add its pin as BACKUP_PIN in a new app release.
// 3. After old cert expires, promote backup to primary.
// 4. Remove old primary pin.
// Alert: pinning failures surface as SSLPeerUnverifiedException — catch and surface
// a user-friendly "update required" state, not a raw crash.

§B-4 — Validate server responses
// Never trust the server blindly — validate status codes and schema
suspend fun <T> safeRequest(call: suspend () -> Response<T>): Result<T> =
    try {
        val response = call()
        when {
            response.isSuccessful && response.body() != null ->
                Result.success(response.body()!!)
            response.code() == 401 ->
                Result.failure(SecurityException("Session expired"))
            response.code() == 403 ->
                Result.failure(SecurityException("Access denied"))
            else ->
                Result.failure(IOException("Unexpected response: ${response.code()}"))
        }
    } catch (e: SSLPeerUnverifiedException) {
        Result.failure(SecurityException("Certificate validation failed", e))
    } catch (e: IOException) {
        Result.failure(e)
    }

§C — Authentication Security
§C-1 — Token lifecycle
// security/auth/SessionManager.kt
class SessionManager(private val prefs: SecurePreferences) {
    companion object {
        private const val KEY_ACCESS_TOKEN  = "access_token"
        private const val KEY_REFRESH_TOKEN = "refresh_token"
        private const val KEY_EXPIRY_MS     = "token_expiry_ms"
    }
    fun saveSession(accessToken: String, refreshToken: String, expiresInSeconds: Long) {
        prefs.putString(KEY_ACCESS_TOKEN,  accessToken)
        prefs.putString(KEY_REFRESH_TOKEN, refreshToken)
        prefs.putString(KEY_EXPIRY_MS, (System.currentTimeMillis() + expiresInSeconds * 1000).toString())
    }
    fun getToken(): String? {
        if (isTokenExpired()) return null
        return prefs.getString(KEY_ACCESS_TOKEN)
    }
    fun getRefreshToken(): String? = prefs.getString(KEY_REFRESH_TOKEN)
    fun isTokenExpired(): Boolean {
        val expiry = prefs.getString(KEY_EXPIRY_MS)?.toLongOrNull() ?: return true
        return System.currentTimeMillis() >= expiry - 30_000L // 30s buffer
    }
    /** Full logout — wipe all session data. Always call server-side revocation too. */
    fun clearSession() {
        prefs.remove(KEY_ACCESS_TOKEN)
        prefs.remove(KEY_REFRESH_TOKEN)
        prefs.remove(KEY_EXPIRY_MS)
    }
}

§C-2 — Biometric authentication (Keystore-backed)
// security/auth/BiometricAuthManager.kt
class BiometricAuthManager(
    private val context: Context,
    private val activity: FragmentActivity,
) {
    sealed class BiometricResult {
        data object Success                    : BiometricResult()
        data object UserCancelled              : BiometricResult()
        data class  Error(val message: String) : BiometricResult()
    }
    private val executor = ContextCompat.getMainExecutor(context)
    /** Prompt for biometric; on success decrypt the Keystore-protected key. */
    fun authenticate(onResult: (BiometricResult) -> Unit) {
        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("Verify identity")
            .setSubtitle("Use biometric to continue")
            .setNegativeButtonText("Cancel")
            .setAllowedAuthenticators(
                BiometricManager.Authenticators.BIOMETRIC_STRONG
            )
            .build()
        BiometricPrompt(activity, executor, object : BiometricPrompt.AuthenticationCallback() {
            override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                onResult(BiometricResult.Success)
            }
            override fun onAuthenticationError(code: Int, msg: CharSequence) {
                if (code == BiometricPrompt.ERROR_USER_CANCELED ||
                    code == BiometricPrompt.ERROR_NEGATIVE_BUTTON
                ) onResult(BiometricResult.UserCancelled)
                else onResult(BiometricResult.Error("Auth error $code: $msg"))
            }
            override fun onAuthenticationFailed() {
                // Single failure — do not surface; prompt will show its own feedback
            }
        }).authenticate(promptInfo)
    }
    companion object {
        fun isAvailable(context: Context): Boolean =
            BiometricManager.from(context)
                .canAuthenticate(BiometricManager.Authenticators.BIOMETRIC_STRONG) ==
                    BiometricManager.BIOMETRIC_SUCCESS
    }
}

§D — Secret Management (API Keys)
D-1 — What NOT to do (and why)
// NEVER — extracted from APK in seconds with apktool / jadx
companion object {
    const val API_KEY    = "sk-live-abc123"    // ← in DEX, readable
    const val SECRET_KEY = "my_secret_value"   // ← same
}
// NEVER — BuildConfig values end up in compiled code
buildConfigField("String", "API_KEY", "\"sk-live-abc123\"") // ← decompilable

D-2 — Correct approach: local.properties + server proxy
// In local.properties (NEVER commit this file)
API_KEY=sk-live-abc123
// In build.gradle.kts — read at build time for non-secret keys only (analytics IDs, etc.)
val apiKey = project.findProperty("API_KEY") as String? ?: ""
buildConfigField("String", "ANALYTICS_ID", "\"$apiKey\"")
// For genuinely sensitive secrets (payment keys, admin tokens):
// → Keep them server-side ONLY.
// → Client requests a short-lived signed token from your backend.
// → Backend validates the client before issuing.
// → Never ship production payment keys in the APK.

D-3 — Runtime secret delivery (recommended pattern)
// Client fetches a short-lived capability token from your own backend
// Backend validates auth + device integrity before issuing
class CapabilityTokenRepository(private val api: CapabilityApi) {
    private var cachedToken: String?  = null
    private var tokenExpiry: Long     = 0L
    suspend fun getToken(): Result<String> {
        if (cachedToken != null && System.currentTimeMillis() < tokenExpiry) {
            return Result.success(cachedToken!!)
        }
        return api.fetchCapabilityToken().map { response ->
            cachedToken  = response.token
            tokenExpiry  = System.currentTimeMillis() + response.expiresInMs
            response.token
        }
    }
}

§E — Encryption
§E-1 — AES-256-GCM with Android Keystore
// security/crypto/AesEncryptor.kt
object AesEncryptor {
    private const val KEY_ALIAS    = "app_aes_key"
    private const val ALGORITHM    = KeyProperties.KEY_ALGORITHM_AES
    private const val BLOCK_MODE   = KeyProperties.BLOCK_MODE_GCM
    private const val PADDING      = KeyProperties.ENCRYPTION_PADDING_NONE
    private const val CIPHER_SPEC  = "$ALGORITHM/$BLOCK_MODE/$PADDING"
    private const val GCM_TAG_BITS = 128
    private fun getOrCreateKey(): SecretKey {
        val keystore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
        keystore.getKey(KEY_ALIAS, null)?.let { return it as SecretKey }
        return KeyGenerator.getInstance(ALGORITHM, "AndroidKeyStore").apply {
            init(
                KeyGenParameterSpec.Builder(
                    KEY_ALIAS,
                    KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
                )
                    .setBlockModes(BLOCK_MODE)
                    .setEncryptionPaddings(PADDING)
                    .setKeySize(256)
                    .setUserAuthenticationRequired(false) // set true + timeout for sensitive keys
                    .build()
            )
        }.generateKey()
    }
    /**
     * Returns: [iv (12 bytes)] + [ciphertext + GCM tag]
     * Store this blob; IV must be unique per encryption — Keystore generates it.
     */
    fun encrypt(plaintext: ByteArray): ByteArray {
        val cipher = Cipher.getInstance(CIPHER_SPEC).apply {
            init(Cipher.ENCRYPT_MODE, getOrCreateKey())
        }
        val ciphertext = cipher.doFinal(plaintext)
        return cipher.iv + ciphertext  // prepend IV
    }
    /**
     * Input: blob produced by encrypt() ([iv] + [ciphertext])
     */
    fun decrypt(blob: ByteArray): ByteArray {
        val iv         = blob.copyOfRange(0, 12)
        val ciphertext = blob.copyOfRange(12, blob.size)
        val cipher = Cipher.getInstance(CIPHER_SPEC).apply {
            init(Cipher.DECRYPT_MODE, getOrCreateKey(), GCMParameterSpec(GCM_TAG_BITS, iv))
        }
        return cipher.doFinal(ciphertext)
    }
}
// Convenience extensions
fun String.encryptToBase64(): String =
    Base64.encodeToString(AesEncryptor.encrypt(toByteArray()), Base64.NO_WRAP)
fun String.decryptFromBase64(): String =
    AesEncryptor.decrypt(Base64.decode(this, Base64.NO_WRAP)).toString(Charsets.UTF_8)

§E-2 — Approved algorithms
Use Case	Algorithm	Key Size
Symmetric encryption	AES-GCM	256-bit
Asymmetric encryption	RSA-OAEP	2048-bit min
Digital signatures	ECDSA (P-256)	—
Key derivation from password	PBKDF2-HMAC-SHA256	≥ 100,000 iterations
Hashing (integrity)	SHA-256 / SHA-3	—
Authenticated encryption (library)	Google Tink	per-primitive
§E-3 — NEVER use
Prohibited	Reason
MD5	Broken — collision attacks trivial
SHA-1	Deprecated for security use — weak
DES / 3DES	Key size too small, Sweet32 attack
AES/ECB	Identical plaintext blocks → identical ciphertext — pattern leakage
RC4	Biased keystream — multiple attacks
Custom cipher	No audit — universally insecure
§F — Input Validation
F-1 — User input
// security/validation/InputValidator.kt
object InputValidator {
    private val EMAIL_REGEX    = Regex("^[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}$")
    private val PHONE_REGEX    = Regex("^\\+?[1-9]\\d{6,14}$")
    private val SAFE_TEXT_REGEX = Regex("^[\\p{L}\\p{N}\\s.,!?'\"()-]{1,500}$")
    fun isEmailValid(email: String): Boolean   = email.length <= 254 && EMAIL_REGEX.matches(email)
    fun isPhoneValid(phone: String): Boolean   = PHONE_REGEX.matches(phone.trim())
    fun isSafeText(text: String): Boolean      = SAFE_TEXT_REGEX.matches(text)
    /** Length guard — always first before any other processing */
    fun enforceMaxLength(input: String, max: Int): String =
        if (input.length > max) throw IllegalArgumentException("Input exceeds maximum length $max") else input
    /** Strip any HTML / script content from user-facing strings */
    fun sanitizeHtml(input: String): String =
        Html.fromHtml(input, Html.FROM_HTML_MODE_COMPACT).toString().trim()
}

F-2 — Intent / deep-link validation
// Validate every incoming Intent extra — do not assume the sending app is trustworthy
override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    val userId = intent.getStringExtra("user_id") ?: return
    if (!userId.matches(Regex("[a-zA-Z0-9_-]{1,64}"))) {
        Timber.w("Rejected malformed user_id from intent")
        return
    }
    // safe to use
}
// Deep link URI validation
fun validateDeepLinkUri(uri: Uri): Boolean {
    val allowedHosts  = setOf("app.example.com", "www.example.com")
    val allowedSchemes = setOf("https", "myapp")
    return uri.scheme in allowedSchemes && uri.host in allowedHosts
}

F-3 — File path validation (path traversal prevention)
// security/validation/FileValidator.kt
object FileValidator {
    private val ALLOWED_MIME_TYPES = setOf("image/jpeg", "image/png", "image/webp", "audio/mp4")
    /**
     * Prevent path traversal: ensure the resolved canonical path
     * is inside the expected directory.
     */
    fun isSafeFilePath(file: File, allowedDir: File): Boolean {
        return try {
            val canonical    = file.canonicalPath
            val allowedCanon = allowedDir.canonicalPath
            canonical.startsWith(allowedCanon + File.separator) || canonical == allowedCanon
        } catch (e: IOException) {
            false
        }
    }
    /** Validate MIME type from ContentResolver — do not trust file extension alone. */
    fun isAllowedMimeType(context: Context, uri: Uri): Boolean {
        val mime = context.contentResolver.getType(uri) ?: return false
        return mime in ALLOWED_MIME_TYPES
    }
    /** Check file size before reading to prevent OOM / DoS. */
    fun isFileSizeAcceptable(context: Context, uri: Uri, maxBytes: Long = 50 * 1024 * 1024): Boolean {
        val size = context.contentResolver.openFileDescriptor(uri, "r")?.use {
            it.statSize
        } ?: return false
        return size <= maxBytes
    }
}

F-4 — Network response validation
// Never use raw server data without schema validation
// With Moshi (strict mode — unknown fields are errors):
val moshi = Moshi.Builder()
    .addLast(KotlinJsonAdapterFactory())
    .build()
// With manual validation
fun parseUserResponse(json: JSONObject): User? {
    val id   = json.optString("id").takeIf { it.isNotBlank() } ?: return null
    val name = json.optString("name").take(200)   // enforce max length even from server
    val role = json.optString("role").takeIf { it in setOf("user", "admin", "viewer") } ?: "user"
    return User(id = id, name = name, role = role)
}

§G — WebView Security
G-1 — Hardened WebView setup
// security/webview/SecureWebViewConfig.kt
object SecureWebViewConfig {
    fun apply(webView: WebView, allowedHost: String) {
        webView.settings.apply {
            // JavaScript — enable ONLY if the app specifically requires it
            javaScriptEnabled    = false     // default off; enable only for trusted content
            // File access — disable to prevent file:// leaks
            allowFileAccess      = false
            allowContentAccess   = false
            // Disable form data saving — prevents credential leak via autofill
            saveFormData         = false
            savePassword         = false
            // Disable geolocation by default
            setGeolocationEnabled(false)
            // Mixed content — strict; never allow HTTP inside HTTPS page
            mixedContentMode     = WebSettings.MIXED_CONTENT_NEVER_ALLOW
        }
        webView.webViewClient = object : WebViewClient() {
            override fun shouldOverrideUrlLoading(view: WebView, request: WebResourceRequest): Boolean {
                val uri = request.url
                // Only allow navigation to trusted host over HTTPS
                val allowed = uri.scheme == "https" && uri.host == allowedHost
                if (!allowed) {
                    Timber.w("WebView blocked navigation to: $uri")
                }
                return !allowed
            }
            override fun onReceivedSslError(view: WebView, handler: SslErrorHandler, error: SslError) {
                // NEVER call handler.proceed() — that disables certificate validation
                handler.cancel()
                Timber.e("WebView SSL error: ${error.primaryError}")
            }
            override fun onReceivedError(
                view: WebView, request: WebResourceRequest, error: WebResourceError
            ) {
                Timber.e("WebView error ${error.errorCode}: ${error.description} for ${request.url}")
            }
        }
    }
    /**
     * Only call if JavaScript is required.
     * Expose the minimum interface; validate every call.
     */
    @SuppressLint("SetJavaScriptEnabled")
    fun enableJavaScript(webView: WebView) {
        webView.settings.javaScriptEnabled = true
    }
    /**
     * JavaScript bridge — keep surface minimal; validate all inputs.
     */
    class MinimalJsBridge(private val onAction: (String) -> Unit) {
        @JavascriptInterface
        fun postAction(action: String) {
            // Validate before acting
            if (action.length > 256 || !action.matches(Regex("[a-zA-Z0-9_]+"))) {
                Timber.w("Rejected invalid JS bridge action: $action")
                return
            }
            onAction(action)
        }
    }
}

§H — Obfuscation (ProGuard / R8)
H-1 — Release build configuration
// app/build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled   = true      // enable R8 shrinking + obfuscation
            isShrinkResources = true      // remove unused resources
            isDebuggable      = false     // NEVER true in release
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            // Reproducible builds — strip timestamps from APK
            packaging.resources.excludes += "META-INF/MANIFEST.MF"
        }
        debug {
            isMinifyEnabled = false
            isDebuggable    = true
        }
    }
}

H-2 — proguard-rules.pro template
# ── Keep data classes used with Gson / Moshi / Kotlinx Serialization ──
-keep class com.yourapp.data.model.** { *; }
# ── Retrofit / OkHttp ──
-keepattributes Signature, InnerClasses, EnclosingMethod
-keepattributes RuntimeVisibleAnnotations, RuntimeVisibleParameterAnnotations
-keepclassmembers,allowshrinking,allowobfuscation interface * {
    @retrofit2.http.* <methods>;
}
-dontwarn org.codehaus.mojo.animal_sniffer.IgnoreJRERequirement
-dontwarn javax.annotation.**
# ── Hilt / Dagger ──
-keep class dagger.hilt.** { *; }
-keep @dagger.hilt.android.HiltAndroidApp class * { *; }
-keepclasseswithmembers class * { @javax.inject.Inject <fields>; }
# ── Room ──
-keep class * extends androidx.room.RoomDatabase { *; }
-keep @androidx.room.Entity class * { *; }
-keep @androidx.room.Dao interface * { *; }
# ── Parcelable / Serializable ──
-keepclassmembers class * implements android.os.Parcelable {
    static ** CREATOR;
}
-keepclassmembers class * implements java.io.Serializable {
    static final long serialVersionUID;
    private static final java.io.ObjectStreamField[] serialPersistentFields;
    private void writeObject(java.io.ObjectOutputStream);
    private void readObject(java.io.ObjectInputStream);
    java.lang.Object writeReplace();
    java.lang.Object readResolve();
}
# ── Enums ──
-keepclassmembers enum * { public static **[] values(); public static ** valueOf(java.lang.String); }
# ── Security — never obfuscate Keystore interaction ──
-keep class android.security.keystore.** { *; }
-keep class javax.crypto.** { *; }
# ── Remove all logging in release ──
-assumenosideeffects class android.util.Log {
    public static *** d(...);
    public static *** v(...);
    public static *** i(...);
    # Keep e() and w() — they may be meaningful in crash reports
}
-assumenosideeffects class timber.log.Timber {
    public static *** d(...);
    public static *** v(...);
}

§I — App Integrity & Tamper Detection
I-1 — Play Integrity API
// security/integrity/IntegrityChecker.kt
// Requires: implementation("com.google.android.play:integrity:1.4.0")
class IntegrityChecker(private val context: Context) {
    sealed class IntegrityResult {
        data object Trusted                            : IntegrityResult()
        data class  Untrusted(val reason: String)      : IntegrityResult()
        data class  Error(val message: String)         : IntegrityResult()
    }
    suspend fun check(): IntegrityResult = withContext(Dispatchers.IO) {
        try {
            val manager = IntegrityManagerFactory.create(context)
            val nonce   = generateNonce() // server-generated for replay protection
            val tokenResponse = Tasks.await(
                manager.requestIntegrityToken(
                    IntegrityTokenRequest.builder()
                        .setNonce(nonce)
                        .build()
                )
            )
            // IMPORTANT: Send token to YOUR backend for verification.
            // Never verify client-side — defeats the purpose.
            val verdict = verifyOnServer(tokenResponse.token())
            if (verdict.deviceRecognized && !verdict.appTampered)
                IntegrityResult.Trusted
            else
                IntegrityResult.Untrusted("Device or app integrity check failed")
        } catch (e: Exception) {
            IntegrityResult.Error("Integrity check failed: ${e.message}")
        }
    }
    private fun generateNonce(): String =
        Base64.encodeToString(
            ByteArray(32).also { SecureRandom().nextBytes(it) },
            Base64.URL_SAFE or Base64.NO_PADDING or Base64.NO_WRAP
        )
    private suspend fun verifyOnServer(token: String): ServerVerdict {
        // POST token to your server; server calls Play Integrity API to decode verdict.
        // This is the only secure verification path.
        TODO("Implement server call")
    }
}

I-2 — Root / tamper detection (supplementary layer)
These checks are heuristics and can be bypassed on sophisticated devices. Always combine with server-side validation. Never gate critical security solely on client-side root detection.

// security/integrity/DeviceIntegrityHeuristics.kt
object DeviceIntegrityHeuristics {
    fun isLikelyCompromised(): Boolean =
        isRooted() || isDebuggable() || isRunningOnEmulator()
    private fun isRooted(): Boolean {
        val superuserPaths = listOf(
            "/system/app/Superuser.apk",
            "/system/xbin/su",
            "/system/bin/su",
            "/sbin/su",
            "/data/local/xbin/su",
        )
        if (superuserPaths.any { File(it).exists() }) return true
        return try {
            Runtime.getRuntime().exec(arrayOf("/system/xbin/which", "su"))
            true
        } catch (e: IOException) {
            false
        }
    }
    private fun isDebuggable(): Boolean =
        Build.TYPE != "user" ||
        (0 != (ApplicationInfo.FLAG_DEBUGGABLE and
            (try { Class.forName("android.app.ActivityThread")
                .getMethod("currentApplication").invoke(null)
                .let { (it as android.app.Application).applicationInfo.flags }
            } catch (e: Exception) { 0 })))
    private fun isRunningOnEmulator(): Boolean =
        Build.FINGERPRINT.startsWith("generic") ||
        Build.FINGERPRINT.startsWith("unknown") ||
        Build.MODEL.contains("Emulator") ||
        Build.MODEL.contains("Android SDK built for x86") ||
        Build.MANUFACTURER.contains("Genymotion") ||
        (Build.BRAND.startsWith("generic") && Build.DEVICE.startsWith("generic")) ||
        Build.PRODUCT == "google_sdk"
}

§J — Secure Logging
J-1 — Timber with production guard
// In Application.onCreate()
if (BuildConfig.DEBUG) {
    Timber.plant(Timber.DebugTree())
} else {
    Timber.plant(ProductionTree()) // send to Crashlytics / custom sink — no PII
}
class ProductionTree : Timber.Tree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        if (priority < Log.WARN) return  // only WARN + ERROR in production
        // Strip any sensitive tokens from message before shipping
        val safeMessage = message.redactSensitivePatterns()
        FirebaseCrashlytics.getInstance().log("[$tag] $safeMessage")
        if (t != null) FirebaseCrashlytics.getInstance().recordException(t)
    }
}
private val SENSITIVE_PATTERNS = listOf(
    Regex("Bearer\\s+[A-Za-z0-9._-]+"),       // JWT / bearer tokens
    Regex("[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}"), // email
    Regex("\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b"), // card number
)
fun String.redactSensitivePatterns(): String =
    SENSITIVE_PATTERNS.fold(this) { acc, regex -> acc.replace(regex, "[REDACTED]") }

J-2 — What is NEVER logged
Category	Examples
Credentials	Passwords, PINs, security answers
Tokens	Access tokens, refresh tokens, session IDs
Keys / secrets	API keys, encryption keys, HMAC secrets
PII	Full name + email together, SSN, passport, card number
Health data	Medical records, biometric data
Stack traces to users	Exception messages shown in UI
§K — Third-Party Dependency Audit
K-1 — Audit checklist
For every dependency:

 Is it actively maintained? (last commit < 6 months ago)
 Does it have known CVEs? (check OSV.dev, ./gradlew dependencyCheckAnalyze)
 Is the maintenance org reputable (Google, Square, JetBrains)?
 Can it be replaced with a first-party Android API?
 Is it actually used? (run ./gradlew :app:dependencies --unused)
 Does it request more permissions than needed in its manifest?
K-2 — OWASP Dependency Check integration
// build.gradle.kts (root)
plugins {
    id("org.owasp.dependencycheck") version "9.2.0"
}
dependencyCheck {
    failBuildOnCVSS    = 7f     // fail on HIGH and CRITICAL
    formats            = listOf("HTML", "JSON")
    suppressionFile    = "config/dependency-check-suppressions.xml"
    nvd.apiKey         = System.getenv("NVD_API_KEY") ?: ""
}
// Run: ./gradlew dependencyCheckAnalyze

K-3 — Version catalog pinning
# gradle/libs.versions.toml — single source of truth for all dependency versions
[versions]
security-crypto  = "1.1.0-alpha06"
okhttp           = "4.12.0"
retrofit         = "2.11.0"
hilt             = "2.51.1"
[libraries]
security-crypto  = { module = "androidx.security:security-crypto",   version.ref = "security-crypto" }
okhttp           = { module = "com.squareup.okhttp3:okhttp",         version.ref = "okhttp" }

§L — Privacy & Data Minimization
L-1 — Manifest permission hygiene
<!-- ONLY declare permissions the app actually uses -->
<!-- Avoid: -->
<uses-permission android:name="android.permission.READ_CONTACTS" />  <!-- if not used -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />  <!-- use COARSE if sufficient -->
<!-- Prefer scoped / coarse alternatives -->
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<!-- Revoke permissions that external SDKs add but you don't need -->
<uses-permission android:name="com.google.android.gms.permission.AD_ID"
    tools:node="remove" />

L-2 — Data minimization rules
// Collect only what the feature strictly requires
// WRONG — collects device ID for analytics that only needs session count
val deviceId = Settings.Secure.getString(contentResolver, Settings.Secure.ANDROID_ID)
analytics.track("session_start", mapOf("device_id" to deviceId))
// CORRECT — anonymous session ID, not tied to device identity
val sessionId = UUID.randomUUID().toString()
analytics.track("session_start", mapOf("session_id" to sessionId))

L-3 — Secure account deletion
// On account deletion: wipe all local data, revoke all tokens
suspend fun deleteAccount(userId: String) {
    // 1. Revoke all server-side tokens
    authApi.revokeAllTokens(userId)
    // 2. Clear encrypted preferences
    securePreferences.clearAll()
    // 3. Delete all user-specific files
    File(context.filesDir, "user_$userId").deleteRecursively()
    // 4. Clear database
    database.clearAllTables()
    // 5. Clear image / media cache
    Coil.imageLoader(context).diskCache?.clear()
    Coil.imageLoader(context).memoryCache?.clear()
}

§M — Security Audit Deliverables Template
Fill this out for every security review or implementation request.

### Security Assessment
[Rate each: Secure / Needs Improvement / Vulnerable / Critical]
- Sensitive data storage:
- Network communication:
- Authentication / session management:
- Secret / key management:
- Input validation:
- Obfuscation / R8:
- Third-party dependencies:
- Privacy compliance:
- Logging hygiene:
### Identified Vulnerabilities
| ID | Vulnerability | Location | Severity | OWASP Mobile Top 10 |
|----|--------------|----------|----------|---------------------|
| V1 | ...          | ...      | Critical/High/Medium/Low | M1–M10 |
### Risk Analysis
For each vulnerability:
- Attack vector: (Local / Network / Physical)
- Exploitability: (Easy / Moderate / Hard)
- Impact: (Data loss / Account takeover / PII exposure / App bypass)
- Affected users: (All / Authenticated / Admin)
### Recommended Mitigations
| ID | Mitigation | Effort | Priority |
|----|-----------|--------|----------|
| V1 | ...       | Hours/Days | P0/P1/P2 |
### Required Code Changes
- File / component to change
- What changes and why
- Code snippet (before → after)
### Compliance Considerations
- [ ] OWASP Mobile Top 10 addressed
- [ ] GDPR / CCPA data minimization applied
- [ ] Play Store policy compliant
- [ ] Android security best practices followed
### Performance Impact
- Expected overhead of security controls added: ...
### Regression Risks
- Existing features affected: ...
- Tests required: ...
### Long-Term Security Roadmap
- Short-term (next sprint): ...
- Medium-term (next quarter): ...
- Long-term (next year): ...

Restrictions (Hard Limits)
The skill must never:

Hardcode secrets, API keys, passwords, or tokens anywhere in source code or resources
Disable or bypass certificate validation for any reason
Use deprecated cryptographic algorithms (MD5, SHA-1, DES, ECB, RC4)
Implement custom cryptographic algorithms
Log passwords, tokens, keys, or PII in any build variant
Store sensitive data in plain SharedPreferences, external storage, or static fields
Expose file:// URIs to external apps (use FileProvider)
Call handler.proceed() in onReceivedSslError (equivalent to trusting all certs)
Add android:debuggable="true" to release builds
Trust client-side integrity checks alone — always pair with server-side validation
Recommend security theater (cosmetic checks) as a substitute for real controls
Sacrifice usability so severely that users circumvent security controls
