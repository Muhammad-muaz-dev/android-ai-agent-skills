Scope
This skill governs internal code quality improvement only. It begins with an audit of the existing code and ends with cleaner, more maintainable, production-ready output that is behaviorally identical to the original.

It does not add features, change business rules, alter navigation, modify API contracts, or touch unrelated modules.

The Prime Directive
Behavior in, behavior out — unchanged.

Every refactoring session must preserve:

All existing business logic
All public APIs and interfaces
All user-visible behavior and UX
All database schemas (unless explicitly requested)
All navigation flows
All existing error handling outcomes
Only internal implementation may change.

How to Use This Skill
Received a refactoring request?
│
├─ Step 1: Audit → §A  (identify smells, risk, scope)
├─ Step 2: Plan  → §L  (produce deliverables before touching code)
│
├─ Then apply relevant fixes:
│   ├─ Naming / readability issues    → §B
│   ├─ Large methods / deep nesting   → §C
│   ├─ God classes / poor cohesion    → §D
│   ├─ SOLID violations               → §E
│   ├─ Non-idiomatic Kotlin           → §F
│   ├─ Dependency / DI problems       → §G
│   ├─ Package structure chaos        → §H
│   ├─ Performance regressions        → §I
│   ├─ Scattered error handling       → §J
│   └─ Missing / broken test coverage → §K
│
└─ Step 3: Deliver → §L  (fill in deliverable template)

Mandatory Rules
#	Rule
R1	Never regenerate or rewrite an entire file unless every line genuinely needs to change. Modify only what is necessary.
R2	Never change business logic, expected outputs, or application flow — even if it looks wrong. Flag it separately.
R3	Never introduce a new abstraction layer (interface, base class, use-case) without a documented justification. Abstractions have a cost.
R4	Never replace stable, working code for stylistic reasons alone. The bar is: will this change make the code measurably easier to maintain, test, or extend?
R5	Always explain every major change — what changed, why, and what risk it carries.
R6	Keep each logical change independently commitable. One concern per change set.
R7	Preserve all existing tests. If a refactoring breaks a test, the refactoring is wrong — not the test.
R8	If a critical path has no tests, recommend tests before refactoring it. Do not silently refactor untested critical paths.
R9	Follow the project's existing architecture, naming conventions, and formatting style — do not impose a foreign style.
R10	When in doubt, prefer the smallest change that provides the most benefit with the least risk.
§A — Pre-Refactoring Audit
Run this audit before writing a single line of refactored code.

A-1 Code Smell Triage Table
For each smell found, classify it: Critical (blocks maintainability), Major (slows development), or Minor (cosmetic).

Smell	Signal	Priority
God Class	Class > 300 lines, > 5 responsibilities	Critical
Long Method	Method > 30 lines, does > 1 thing	Critical
Deep Nesting	if/for/when > 3 levels deep	Critical
Duplicate Code	Same logic copy-pasted in ≥ 2 places	Critical
Feature Envy	Method heavily uses another class's data	Major
Primitive Obsession	Raw String/Int used where a value class fits	Major
Excessive Parameters	Function with > 4 parameters	Major
Magic Numbers/Strings	Unexplained literals (if (status == 3))	Major
Dead Code	Unreachable code, unused functions, unused imports	Major
Tight Coupling	class A { val b = B() } — concrete instantiation	Major
Cyclic Dependency	Package A imports package B imports package A	Critical
Ambiguous Names	temp, data, helper, obj, manager2	Minor
Stale Comments	Comment contradicts the code it describes	Minor
Redundant Conditions	if (x == true), if (list != null && list.isNotEmpty())	Minor
A-2 Architecture Compliance Audit
Verify dependency direction before refactoring. In Clean Architecture:

UI (Fragment/Activity)
    → ViewModel
        → UseCase / Domain
            → Repository interface
                ← Repository implementation (data layer)
                    → DataSource (local / remote)

Flag any violation:

Violation	Example	Fix
UI contains business logic	Fragment formats a price string	Move to ViewModel or mapper
ViewModel calls Retrofit directly	vm.apiService.getUser()	Add Repository
UseCase touches UI/Android SDK	UseCase holds Context	Remove; pass primitives instead
Repository contains UI logic	Repository updates LiveData	Return domain model; let VM map
Cross-feature direct imports	featureA imports featureB.ui.*	Communicate via shared domain / events
A-3 Impact & Risk Assessment
Before starting, answer:

Which public interfaces will change? (If any, document the before/after contract explicitly.)
Which existing tests cover the code being changed?
What is the shortest change that delivers the most benefit?
Are there any untested critical paths? (Recommend tests first.)
§B — Naming & Readability
B-1 Naming Rules
Identifier	Convention	Bad Example	Good Example
Class	UpperCamelCase, noun	DataHelper, MyManager	UserRepository, LoginViewModel
Function	lowerCamelCase, verb	doStuff(), process()	fetchUserProfile(), validateInput()
Variable	lowerCamelCase, noun	tmp, d, obj	userProfile, recordingDurationMs
Boolean	is/has/can prefix	active, loggedIn	isActive, isUserLoggedIn
Constants	UPPER_SNAKE_CASE	maxSize, defaultVal	MAX_RETRY_COUNT, DEFAULT_TIMEOUT_MS
Packages	lowercase.dotted	Utils, MyHelpers	com.app.feature.login.domain
B-2 Names to always replace
// BEFORE — meaningless
val temp = getUserFromDb()
val data = processResult(temp)
fun helper(x: String): Boolean { ... }
class Manager { ... }
// AFTER — self-explanatory
val cachedUser      = getUserFromDb()
val processedResult = processResult(cachedUser)
fun isEmailValid(email: String): Boolean { ... }
class AudioPlaybackController { ... }

B-3 Magic literal elimination
// BEFORE
if (response.code == 401) { logout() }
delay(3000)
val maxItems = 50
// AFTER
private const val HTTP_UNAUTHORIZED    = 401
private const val RETRY_DELAY_MS       = 3_000L
private const val MAX_FEED_ITEMS       = 50
if (response.code == HTTP_UNAUTHORIZED) { logout() }
delay(RETRY_DELAY_MS)
val maxItems = MAX_FEED_ITEMS

B-4 Comment strategy
// REMOVE — restates the code
// increment i by 1
i++
// REMOVE — stale / inaccurate
// TODO: fix this someday (written 3 years ago, the bug was fixed)
// KEEP — explains WHY, not what
// Delay required: the hardware sensor needs 200 ms to stabilize after wake.
delay(200)
// KEEP — flags a non-obvious contract
// Must be called on the main thread — SpeechRecognizer is not thread-safe.
recognizer.startListening(intent)

§C — Method Refactoring
C-1 Break large methods — Extract Function pattern
// BEFORE — one method doing five things (45 lines condensed)
fun processOrder(order: Order) {
    // validate
    if (order.items.isEmpty()) throw IllegalArgumentException("empty")
    if (order.userId.isBlank()) throw IllegalArgumentException("no user")
    // calculate
    val subtotal = order.items.sumOf { it.price * it.quantity }
    val tax      = subtotal * 0.2
    val total    = subtotal + tax
    // apply discount
    val discount = if (order.couponCode == "SAVE10") total * 0.1 else 0.0
    val finalTotal = total - discount
    // format receipt
    val receipt = "Order #${order.id}\nTotal: $${"%.2f".format(finalTotal)}"
    // save
    db.save(order.copy(total = finalTotal))
    analytics.track("order_placed", mapOf("total" to finalTotal))
}
// AFTER — each method does one thing
fun processOrder(order: Order) {
    validateOrder(order)
    val finalTotal = calculateOrderTotal(order)
    val receipt    = formatReceipt(order, finalTotal)
    persistOrder(order, finalTotal)
}
private fun validateOrder(order: Order) {
    require(order.items.isNotEmpty()) { "Order must contain at least one item" }
    require(order.userId.isNotBlank()) { "Order must be associated with a user" }
}
private fun calculateOrderTotal(order: Order): Double {
    val subtotal = order.items.sumOf { it.price * it.quantity }
    val tax      = subtotal * TAX_RATE
    val discount = applyCouponDiscount(subtotal + tax, order.couponCode)
    return subtotal + tax - discount
}
private fun applyCouponDiscount(total: Double, couponCode: String?): Double =
    if (couponCode == VALID_COUPON_CODE) total * COUPON_DISCOUNT_RATE else 0.0
private fun formatReceipt(order: Order, total: Double): String =
    "Order #${order.id}\nTotal: $${"%.2f".format(total)}"
private fun persistOrder(order: Order, total: Double) {
    db.save(order.copy(total = total))
    analytics.track("order_placed", mapOf("total" to total))
}

C-2 Reduce deep nesting — Early Return / Guard Clause
// BEFORE — arrow anti-pattern
fun getDisplayName(user: User?): String {
    if (user != null) {
        if (user.profile != null) {
            if (user.profile.displayName != null) {
                if (user.profile.displayName.isNotBlank()) {
                    return user.profile.displayName
                }
            }
        }
    }
    return "Anonymous"
}
// AFTER — flat with guard clauses
fun getDisplayName(user: User?): String {
    val displayName = user?.profile?.displayName
    if (displayName.isNullOrBlank()) return "Anonymous"
    return displayName
}

C-3 Replace long if-else chains with when
// BEFORE
fun describeStatus(code: Int): String {
    if (code == 200) return "OK"
    else if (code == 201) return "Created"
    else if (code == 400) return "Bad Request"
    else if (code == 401) return "Unauthorized"
    else if (code == 403) return "Forbidden"
    else if (code == 404) return "Not Found"
    else if (code == 500) return "Server Error"
    else return "Unknown ($code)"
}
// AFTER
fun describeStatus(code: Int): String = when (code) {
    200  -> "OK"
    201  -> "Created"
    400  -> "Bad Request"
    401  -> "Unauthorized"
    403  -> "Forbidden"
    404  -> "Not Found"
    500  -> "Server Error"
    else -> "Unknown ($code)"
}

C-4 Eliminate duplicate logic — Extract shared function
// BEFORE — same null-check + log pattern in 6 places
fun saveUser(user: User?) {
    if (user == null) {
        Log.e(TAG, "saveUser: user is null")
        return
    }
    db.save(user)
}
fun deleteUser(user: User?) {
    if (user == null) {
        Log.e(TAG, "deleteUser: user is null")
        return
    }
    db.delete(user)
}
// AFTER — extracted guard
private inline fun withValidUser(user: User?, action: String, block: (User) -> Unit) {
    if (user == null) {
        Log.e(TAG, "$action called with null user")
        return
    }
    block(user)
}
fun saveUser(user: User?)   = withValidUser(user, "saveUser")   { db.save(it) }
fun deleteUser(user: User?) = withValidUser(user, "deleteUser") { db.delete(it) }

C-5 Reduce parameter count — Parameter Object pattern
// BEFORE — 7 parameters, hard to read at call site
fun scheduleNotification(
    userId: String, title: String, body: String,
    scheduledAt: Long, channelId: String,
    priority: Int, autoCancel: Boolean
)
// AFTER — group related parameters into a value class / data class
data class NotificationRequest(
    val userId:      String,
    val title:       String,
    val body:        String,
    val scheduledAt: Long,
    val channelId:   String,
    val priority:    Int    = NotificationCompat.PRIORITY_DEFAULT,
    val autoCancel:  Boolean = true,
)
fun scheduleNotification(request: NotificationRequest)

§D — Class & Architecture Refactoring
D-1 Split a God Class
BEFORE: UserManager (600 lines)
  ├── login / logout / session         → AuthRepository
  ├── fetch / cache user profile       → UserRepository
  ├── format display name / avatar URL → UserMapper
  ├── analytics events                 → AnalyticsService (inject)
  └── push token registration         → PushTokenRepository

// Split target — extract each responsibility to its own class
// Keep UserManager as a thin facade only if callers cannot be updated immediately.
class AuthRepository(private val api: AuthApi, private val prefs: SecurePrefs) {
    suspend fun login(credentials: Credentials): Result<Session> { ... }
    fun logout() { ... }
    fun isSessionValid(): Boolean { ... }
}
class UserRepository(private val api: UserApi, private val db: UserDao) {
    suspend fun getProfile(userId: String): Result<UserProfile> { ... }
    suspend fun updateProfile(update: ProfileUpdate): Result<Unit> { ... }
}
object UserMapper {
    fun toDisplayModel(profile: UserProfile): UserDisplayModel { ... }
}

D-2 Enforce Single Responsibility — tell-tale signs
A class violates SRP if you cannot complete this sentence in one clause:
_"This class is responsible for __."

// VIOLATION — two responsibilities in one class
class LoginViewModel(
    private val api: AuthApi,   // ← data fetching
    private val prefs: SharedPreferences, // ← persistence
) : ViewModel() {
    fun login(email: String, password: String) {
        // calls API directly — should go through Repository
        viewModelScope.launch {
            val response = api.login(email, password) // ← VM doing data work
            prefs.edit().putString("token", response.token).apply()
        }
    }
}
// FIXED — ViewModel delegates, Repository owns data
class LoginViewModel(
    private val loginUseCase: LoginUseCase,
) : ViewModel() {
    fun login(email: String, password: String) {
        viewModelScope.launch { loginUseCase(email, password) }
    }
}

D-3 Primitive Obsession — replace with value classes
// BEFORE — raw primitives; easy to pass in wrong order
fun transferFunds(fromAccountId: String, toAccountId: String, amountCents: Int)
// AFTER — domain types; compiler enforces correct usage
@JvmInline value class AccountId(val value: String)
@JvmInline value class Cents(val value: Int) {
    init { require(value >= 0) { "Amount cannot be negative" } }
}
fun transferFunds(from: AccountId, to: AccountId, amount: Cents)

D-4 Feature Envy — move the method to where the data lives
// BEFORE — OrderPrinter reaches deep into Order's internals
class OrderPrinter {
    fun printSummary(order: Order): String {
        val total = order.items.sumOf { it.price * it.quantity }
        return "Order ${order.id}: $${"%.2f".format(total)}"
    }
}
// AFTER — move the logic to Order (or its mapper) where the data lives
data class Order(val id: String, val items: List<OrderItem>) {
    val totalCents: Int get() = items.sumOf { it.priceCents * it.quantity }
    val formattedTotal: String get() = "$${"%.2f".format(totalCents / 100.0)}"
}
class OrderPrinter {
    fun printSummary(order: Order): String = "Order ${order.id}: ${order.formattedTotal}"
}

§E — SOLID Principles
E-1 Single Responsibility (see §D-1, §D-2)
E-2 Open/Closed — extend behavior without modifying existing classes
// BEFORE — adding a payment method requires editing PaymentProcessor
class PaymentProcessor {
    fun process(type: String, amount: Double) {
        when (type) {
            "card"   -> processCard(amount)
            "paypal" -> processPaypal(amount)
            // Must edit here every time a new method is added ← violation
        }
    }
}
// AFTER — new payment methods extend; core class never changes
interface PaymentHandler {
    fun process(amount: Double)
}
class CardPaymentHandler(private val gateway: CardGateway) : PaymentHandler {
    override fun process(amount: Double) = gateway.charge(amount)
}
class PaypalPaymentHandler(private val client: PaypalClient) : PaymentHandler {
    override fun process(amount: Double) = client.pay(amount)
}
class PaymentProcessor(private val handler: PaymentHandler) {
    fun process(amount: Double) = handler.process(amount)
}

E-3 Liskov Substitution — subtypes must honor the supertype contract
// VIOLATION — Square breaks Rectangle's contract
open class Rectangle {
    open var width: Int = 0
    open var height: Int = 0
    fun area() = width * height
}
class Square : Rectangle() {
    override var width: Int
        set(value) { super.width = value; super.height = value } // ← breaks area() for callers
    // ...
}
// FIX — model the domain correctly; don't force an inheritance that doesn't fit
sealed class Shape {
    data class Rectangle(val width: Int, val height: Int) : Shape()
    data class Square(val side: Int) : Shape()
}
fun Shape.area(): Int = when (this) {
    is Shape.Rectangle -> width * height
    is Shape.Square    -> side * side
}

E-4 Interface Segregation — narrow interfaces
// BEFORE — clients forced to implement methods they don't use
interface MediaController {
    fun play()
    fun pause()
    fun record()    // audio-only clients don't record video
    fun capture()   // camera-only clients don't play audio
}
// AFTER — segregated
interface Playable  { fun play();    fun pause() }
interface Recordable { fun record(); fun stop()  }
interface Capturable { fun capture() }
class AudioPlayer  : Playable { ... }
class AudioRecorder : Recordable { ... }
class Camera        : Capturable { ... }

E-5 Dependency Inversion — depend on abstractions
// BEFORE — high-level policy depends on low-level detail
class ReportGenerator {
    private val db = MySqlDatabase() // ← concrete, cannot be swapped or tested
    fun generate(id: String) = db.query("SELECT * FROM reports WHERE id = '$id'")
}
// AFTER — depend on an abstraction; inject the implementation
interface ReportDataSource {
    suspend fun fetchReport(id: String): Report
}
class ReportGenerator(private val dataSource: ReportDataSource) {
    suspend fun generate(id: String): Report = dataSource.fetchReport(id)
}
// Test freely without a real database
class FakeReportDataSource : ReportDataSource {
    override suspend fun fetchReport(id: String) = Report(id, "Test Report")
}

§F — Kotlin-Specific Modernization
Apply these only where they improve clarity. Never sacrifice readability for brevity.

F-1 Replace Java-style null checks with idiomatic Kotlin
// BEFORE
if (user != null) {
    if (user.name != null && !user.name!!.isEmpty()) {
        textView.text = user.name!!
    } else {
        textView.text = "Anonymous"
    }
}
// AFTER
textView.text = user?.name?.takeIf { it.isNotBlank() } ?: "Anonymous"

F-2 Replace apply / also / let / run idiomatically
// BEFORE — verbose builder pattern
val paint = Paint()
paint.color = Color.RED
paint.strokeWidth = 4f
paint.isAntiAlias = true
// AFTER — apply for object configuration
val paint = Paint().apply {
    color        = Color.RED
    strokeWidth  = 4f
    isAntiAlias  = true
}

// BEFORE — repetitive null checks
val result = fetchData()
if (result != null) {
    process(result)
    log(result)
}
// AFTER — let for null-guarded blocks
fetchData()?.let { result ->
    process(result)
    log(result)
}

F-3 Replace enum with sealed class where states carry data
// BEFORE — enum cannot carry different data per variant
enum class NetworkState { LOADING, SUCCESS, ERROR }
// AFTER — sealed class carries typed data
sealed class NetworkState {
    data object Loading                              : NetworkState()
    data class  Success<T>(val data: T)             : NetworkState()
    data class  Error(val message: String,
                      val code: Int? = null)        : NetworkState()
}

F-4 Use data class + copy() for immutable state updates
// BEFORE — mutable state, easy to introduce race conditions
class UiState {
    var isLoading: Boolean = false
    var userName: String = ""
    var errorMessage: String? = null
}
// AFTER — immutable; update with copy()
data class UiState(
    val isLoading:    Boolean = false,
    val userName:     String  = "",
    val errorMessage: String? = null,
)
// In ViewModel
_uiState.update { it.copy(isLoading = true) }

F-5 Replace for loops with collection functions
// BEFORE
val activeUsers = mutableListOf<User>()
for (user in users) {
    if (user.isActive) {
        activeUsers.add(user)
    }
}
val names = mutableListOf<String>()
for (user in activeUsers) {
    names.add(user.displayName)
}
// AFTER
val names = users
    .filter { it.isActive }
    .map    { it.displayName }

F-6 Use Flow over callbacks for async streams
// BEFORE — callback-based API, hard to compose or test
interface LocationCallback {
    fun onLocationUpdate(lat: Double, lng: Double)
    fun onError(error: String)
}
// AFTER — Flow; composable, testable, lifecycle-aware
fun observeLocation(): Flow<Location> = callbackFlow {
    val listener = object : LocationListener {
        override fun onLocationChanged(loc: Location) { trySend(loc) }
        override fun onError(msg: String)             { close(RuntimeException(msg)) }
    }
    locationManager.requestUpdates(listener)
    awaitClose { locationManager.removeUpdates(listener) }
}

§G — Dependency & Injection Improvements
G-1 Remove manual object construction from classes
// BEFORE — hard to test; cannot swap implementations
class LoginViewModel : ViewModel() {
    private val api  = RetrofitClient.create(AuthApi::class.java)  // ← manual
    private val prefs = SharedPreferencesManager(App.context)      // ← global state
}
// AFTER — constructor-injected; Hilt / Koin provides at runtime, Fake in tests
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val loginUseCase:  LoginUseCase,
    private val sessionStore:  SessionStore,
) : ViewModel()

G-2 Hilt module structure
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides @Singleton
    fun provideOkHttpClient(): OkHttpClient =
        OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .build()
    @Provides @Singleton
    fun provideAuthApi(client: OkHttpClient): AuthApi =
        Retrofit.Builder()
            .baseUrl(BuildConfig.API_BASE_URL)
            .client(client)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(AuthApi::class.java)
}
@Module
@InstallIn(ViewModelComponent::class)
object UseCaseModule {
    @Provides
    fun provideLoginUseCase(repo: AuthRepository): LoginUseCase = LoginUseCase(repo)
}

G-3 Detect and fix cyclic dependencies
SYMPTOM: Hilt or Dagger reports circular dependency at compile time.
CAUSE:   ClassA → ClassB → ClassA (directly or transitively)
FIX OPTIONS (in order of preference):
1. Extract the shared concern to ClassC that both A and B depend on.
2. Pass data as constructor parameters instead of injecting the full class.
3. Use `Lazy<ClassB>` injection to break the initialization cycle.

§H — Package Organization
H-1 Feature-first package structure (preferred)
com.app
 ├── core/
 │    ├── network/        ← Retrofit, OkHttp setup
 │    ├── database/       ← Room, DAO base classes
 │    ├── di/             ← App-level Hilt modules
 │    └── util/           ← True utilities (extension fns, formatters)
 ├── feature/
 │    ├── login/
 │    │    ├── data/      ← LoginRepository, LoginApi, LoginDao
 │    │    ├── domain/    ← LoginUseCase, LoginResult (domain model)
 │    │    ├── presentation/ ← LoginViewModel, LoginFragment, LoginUiState
 │    │    └── di/        ← LoginModule
 │    ├── feed/
 │    │    └── ...
 │    └── profile/
 │         └── ...
 └── shared/
      ├── domain/         ← Shared domain models (User, Session)
      └── ui/             ← Shared UI components (LoadingView, ErrorView)

H-2 Move checklist
Before moving a file:

 Update all import statements in affected files
 Verify no circular imports are created
 Confirm DI modules reference the new path
 Run full build + tests after each batch of moves
 Do not mix package moves with logic changes — one PR per concern
§I — Performance-Safe Refactoring
Every refactoring must be verified to not degrade performance. These patterns improve both quality and performance.

I-1 Remove unnecessary allocations
// BEFORE — new lambda allocated on every bind
override fun onBindViewHolder(holder: ViewHolder, position: Int) {
    holder.button.setOnClickListener { onItemClick(getItem(position)) } // ← allocation
}
// AFTER — listener set once in init; captures position at call time
inner class ViewHolder(binding: ItemBinding) : RecyclerView.ViewHolder(binding.root) {
    init {
        binding.button.setOnClickListener {
            val pos = bindingAdapterPosition
            if (pos != RecyclerView.NO_ID) onItemClick(getItem(pos))
        }
    }
}

I-2 Lazy initialization for expensive objects
// BEFORE — initialized at class construction even if never used
class AnalyticsHelper {
    private val heavyTracker = HeavyAnalyticsTracker() // always constructed
}
// AFTER — initialized on first access
class AnalyticsHelper {
    private val heavyTracker by lazy { HeavyAnalyticsTracker() }
}

I-3 Replace repeated collection copies with sequences
// BEFORE — 3 intermediate lists created
val result = items
    .filter  { it.isActive }      // List<Item>
    .map     { it.toDisplayModel() } // List<DisplayModel>
    .take(10)                      // List<DisplayModel>
// AFTER — zero intermediate collections (sequence is lazy)
val result = items
    .asSequence()
    .filter  { it.isActive }
    .map     { it.toDisplayModel() }
    .take(10)
    .toList()

I-4 Move computation out of draw / bind methods
// BEFORE — formatting on every bind
fun bind(item: Item) {
    binding.price.text = NumberFormat.getCurrencyInstance().format(item.priceCents / 100.0)
    //                   ↑ creates a new NumberFormat instance every bind call
}
// AFTER — pre-format in the mapper before submitList
data class ItemDisplayModel(
    val id: String,
    val formattedPrice: String, // formatted once, at mapping time
)
fun bind(model: ItemDisplayModel) {
    binding.price.text = model.formattedPrice
}

§J — Error Handling Consolidation
J-1 Replace scattered try-catch with centralized mapping
// BEFORE — identical catch blocks in 10 repository functions
suspend fun getUser(id: String): User? {
    return try {
        api.getUser(id)
    } catch (e: HttpException) {
        Log.e(TAG, "HTTP error: ${e.code()}")
        null
    } catch (e: IOException) {
        Log.e(TAG, "IO error: ${e.message}")
        null
    }
}
// AFTER — one extension; call sites become one-liners
suspend fun <T> safeApiCall(call: suspend () -> T): Result<T> =
    try {
        Result.success(call())
    } catch (e: HttpException) {
        Result.failure(ApiException.Http(e.code(), e.message()))
    } catch (e: IOException) {
        Result.failure(ApiException.Network(e.message ?: "Network error"))
    } catch (e: Exception) {
        Result.failure(ApiException.Unknown(e.message ?: "Unexpected error"))
    }
sealed class ApiException(message: String) : Exception(message) {
    class Http(val code: Int, msg: String)  : ApiException("HTTP $code: $msg")
    class Network(msg: String)              : ApiException("Network: $msg")
    class Unknown(msg: String)              : ApiException(msg)
}
// Usage — clean and consistent
suspend fun getUser(id: String): Result<User> = safeApiCall { api.getUser(id) }

J-2 Never suppress exceptions silently
// NEVER DO THIS
try {
    riskyOperation()
} catch (e: Exception) {
    // swallowed — no log, no state update, nothing
}
// ALWAYS — log and surface a typed state
try {
    riskyOperation()
} catch (e: Exception) {
    Timber.e(e, "riskyOperation failed")
    _state.update { it.copy(error = "Operation failed: ${e.message}") }
}

§K — Testing Awareness
K-1 Assess coverage before refactoring
For each piece of code targeted for refactoring, check:
□ Are there unit tests for this class/function?
□ Are there integration tests that exercise this path?
□ Is this a critical path (auth, payment, data persistence)?
If critical path + no tests → write tests FIRST, then refactor.
If non-critical + no tests  → refactor + add tests as part of the same change.
If tests exist              → run them before and after; they must pass unchanged.

K-2 Testability improvements from refactoring
After refactoring, code should be testable without:

A real database
A real network
The Android framework (where possible)
Application or Context (pass only what you need)
// BEFORE — untestable; Context deeply embedded
class UserValidator(private val context: Context) {
    fun isEmailValid(email: String): Boolean {
        val pattern = context.getString(R.string.email_regex)
        return email.matches(pattern.toRegex())
    }
}
// AFTER — testable anywhere; no Context needed
object UserValidator {
    private val EMAIL_REGEX = Regex("^[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}$")
    fun isEmailValid(email: String): Boolean = email.matches(EMAIL_REGEX)
}

K-3 Recommended test types per layer
Layer	Test type	Framework
Domain / UseCase	Unit test	JUnit 5, Turbine (Flow)
Repository	Unit test with fake DataSource	JUnit 5, MockK
ViewModel	Unit test	JUnit 5, Turbine, TestCoroutineScheduler
Adapter / Mapper	Unit test	JUnit 5
Fragment / Activity	Instrumented	Espresso, Hilt testing
End-to-end	Instrumented	UI Automator
§L — Deliverables Template
Fill this out before writing refactored code, and deliver it alongside the changes.

### Code Quality Assessment
[Rate each: Good / Needs Improvement / Critical]
- Readability:
- Naming conventions:
- Method size:
- Class cohesion:
- Architecture compliance:
- Test coverage:
### Identified Code Smells
| Smell | Location | Priority |
|-------|----------|----------|
| ...   | ...      | ...      |
### Refactoring Objectives
1. ...
2. ...
### Files Modified
| File | Change Summary | Risk |
|------|---------------|------|
| ...  | ...           | Low/Medium/High |
### Regression Risk Assessment
- Functions with behavior changes: None (or list them)
- Tests that must pass after change: (list)
- Manual verification steps: (list)
### Performance Considerations
- No performance regressions expected because: ...
- Any improvements: ...
### Maintainability Improvements
- ...
### Testing Recommendations
- ...

Restrictions (Hard Limits)
The skill must never:

Change business logic, expected outputs, or application flow
Add new features or screens
Remove existing functionality
Rewrite unrelated files or modules not named in scope
Introduce new abstraction layers without documented justification
Break or modify public API contracts
Change database schemas unless explicitly requested
Replace stable, working code for purely stylistic reasons
Increase complexity in the name of "proper architecture"
Suppress or swallow exceptions
Silently refactor untested critical paths — flag missing tests first
Regenerate entire files when only a section needs changing
Mix package/file moves with logic changes in the same change set
