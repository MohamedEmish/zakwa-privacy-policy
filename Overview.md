# ARCHITECTURE

This document describes how the Zakwa app is put together at runtime: how features compose, how data flows, how the app stays in sync, and how each platform launches the shared Kotlin code.

---

## 1. What Zakwa does

Zakwa is a personal **Zakat calculator** for Muslims. The user records their wealth (gold, silver, cash) in multiple currencies; the app continuously tracks the **gold and silver Nisab thresholds** using live market prices, computes Zakat obligations once wealth crosses the threshold, and reminds the user when annual dues come up.

Two product invariants shape every architectural choice:

1. **Wealth data never leaves the device.** The app fetches gold/silver prices and currency exchange rates from public APIs, but a user's wealth list, debts, and calculated Zakat are stored only in the encrypted local DataStore + Room database.
2. **Offline-first.** Prices and history are cached aggressively; calculations run against the local cache. Background sync refreshes the cache when connectivity is available.

---

## 2. Tech stack at a glance

| Layer            | Tech                                                           |
| ---------------- | -------------------------------------------------------------- |
| UI               | Compose Multiplatform 1.11.x (Material 3)                      |
| State            | Custom MVI on `BaseViewModel` (StateFlow + Channel)            |
| DI               | Koin (Compose ViewModel integration)                           |
| Networking       | Ktor client + Ktorfit (KSP-generated typed stubs)              |
| Local DB         | Room KMP 2.8.x (`PriceHistoryEntity`, `WealthEntity`, `ZakatEntity`) |
| Key-value store  | androidx.datastore (encrypted, platform-specific factories)    |
| Charts           | Custom Compose-drawn line chart in `cmpCore:ui`                |
| Animations       | Compottie (Lottie for Compose) via `SimpleLabLottieAnimation`  |
| Background work  | WorkManager (Android), BGAppRefresh (iOS), Timer (Desktop)     |
| Crash reporting  | Firebase Crashlytics + Kermit (logger)                         |
| Build            | Gradle 9.x + AGP 9.x with `build-logic` convention plugins     |

---

## 3. Module layering

The repo has three concentric layers — **app modules** (one per platform), the **assembly modules** that ferry shared code to each platform, and the **core/feature** modules that hold the actual logic and UI.

```
                  ┌──────────────┐  ┌─────────────┐  ┌──────────────┐
   Platform apps  │  cmp-android │  │  cmp-ios *  │  │  cmp-desktop │
                  └──────┬───────┘  └──────┬──────┘  └──────┬───────┘
                         │                 │                │
                         └──────────► cmp-shared ◄──────────┘
                                           │
                                     cmp-navigation
                                           │
                ┌────────────┬─────────────┴─────────────┬─────────┐
           feature/home   feature/wealth         feature/zakat   feature/more
                                           │
                         ┌─────────────cmpCore─────────────┐
                         │ui             data     domain   │
                         │model         common   network   │
                         │database     datastore           │
                         │platform     analytics    sync   │
                         └─────────────────────────────────┘

   * cmp-ios is an Xcode workspace under cmp-ios/, not a Gradle module.
```

**Direction rule:** features depend on cores; cores never depend on features. Within `cmpCore`, the direction is always "down" — for instance `:cmpCore:data` may depend on `:cmpCore:network`, `:database`, `:domain`, `:model`; `:cmpCore:domain` only depends on `:common`/`:model`.

`cmp.feature.convention` automatically adds `:cmpCore:ui`, `:cmpCore:data`, and `:cmpCore:analytics` to every feature module (and to `cmp-navigation` / `cmp-shared` which apply it too), so you won't see these declared explicitly in those `build.gradle.kts` files — that's by design after the Phase 2 dependency cleanup.

---

## 4. Navigation

The navigation graph is defined entirely in `cmp-navigation` using **Jetpack Navigation Compose** (multiplatform port). The shape is:

```
RootNavGraph
├── Splash                  cmp-navigation/.../splash/SplashScreen.kt
└── AuthenticatedGraph      cmp-navigation/.../authenticatednavbar/
    ├── HomeGraph           feature/home/
    │   ├── HomeScreen
    │   └── ChartsScreen
    ├── WealthGraph         feature/wealth/
    │   ├── WealthListScreen
    │   └── EditWealthScreen
    ├── ZakatGraph          feature/zakat/
    │   └── ZakatListScreen
    └── MoreGraph           feature/more/
        ├── MoreScreen
        ├── UserProfileScreen
        └── PreferencesScreen
```

The bottom navigation bar lives in `cmp-navigation/.../authenticatednavbar/AuthenticatedNavbarScreen.kt`. Each tab graph is contributed by its feature module via an extension function on `NavGraphBuilder` (search for `fun NavGraphBuilder.homeGraph` etc.). To add a new screen, add an entry in the appropriate feature's `navigation/` package, then wire it into the parent graph.

Routes are typed sealed objects (not raw strings) — see `HOME_SCREEN_ROUTE`-style consts and the `KmpNavigation` annotation marker.

---

## 5. Data persistence

### Room database (`cmpCore:database`)

| Entity              | Purpose                                                    | Key fields                                                |
| ------------------- | ---------------------------------------------------------- | --------------------------------------------------------- |
| `WealthEntity`      | User-recorded assets (gold, silver, money) and debts.      | `zakatType`, `pureMetalValue`, `currency`, `isDebt`, `isDeleted`, `lastUpdatedAt` |
| `ZakatEntity`       | Calculated Zakat obligations.                              | `dueAt`, `dueFor` (year), `appMode`, `isDeleted`          |
| `PriceHistoryEntity`| Daily gold/silver price snapshots — feeds the chart.       | `goldPrice`, `silverPrice`, `currency`, `createdAt` (indexed unique per day) |

DAOs live alongside in `cmpCore/database/src/commonMain/kotlin/.../dao/`. Note: `WealthEntity` and `ZakatEntity` use **soft-delete** via `isDeleted` — query helpers should always filter on `WHERE isDeleted = 0` unless you're explicitly restoring or purging.

### DataStore (`cmpCore:datastore`)

Encrypted DataStore stores everything else, keyed via string Preferences. Notable keys:

- `mainCurrency`, `appTheme`, `appMode`, `language` — user settings.
- `goldPriceDto`, `silverPriceDto`, `currencyExchangeDto` — JSON-serialized latest snapshots from the Firebase Functions endpoint.
- `dueMonth`, `dueDayOfMonth` — when the user's next Zakat is due.
- `notificationId` — monotonic counter for system notifications.
- `historySyncCursor` — Unix-seconds timestamp of the most recent price history page successfully ingested. Drives incremental `getPriceHistory(after:)` calls.

The `:platform` module provides the `expect`/`actual` DataStore factory and any other platform-specific glue (file paths, Android `Context`, NSUserDefaults, etc.).

---

## 6. External APIs

The app talks to three HTTP services. All are typed via Ktorfit interfaces under `cmpCore/network/src/commonMain/kotlin/.../api/`.

### a. Zakwa Firebase Functions

The primary backend — a thin Firebase Functions wrapper (Node 22 source under `functions/`) that aggregates gold, silver, and exchange-rate data into one payload and exposes price history pagination.

| Endpoint              | Returns                                                           |
| --------------------- | ----------------------------------------------------------------- |
| `GET /getLatestData`  | Latest gold + silver + currency-exchange snapshots in one bundle. |
| `GET /getPriceHistory?after={seconds}` | Page of historical gold/silver price points after the given Unix-seconds cursor. |

`MarketRepositoryImpl.appConfigurationFlow` calls `getLatestData` and persists each field into DataStore. `syncMissingHistory()` loops `getPriceHistory` paginated by the `historySyncCursor` until the server returns an empty page.

### b. Gold API (`GoldApiService`)

Public spot-price endpoint, used as a secondary/fallback source. `GET /api/{symbol}/{currency}` where `symbol ∈ {GOLD, SILVER}`. Returns a single `MetalPriceNetworkResponse`.

### c. Currency Exchange API (`CurrencyExchangeApiService`)

`GET {date}?base={currency}` returns the day's exchange rate map keyed by ISO currency code. Used when the user's `mainCurrency` differs from the API's default (`DEFAULT_CURRENCY`).

All three services return wrapped responses via the `NetworkResult<T, NetworkError>` type — the repository layer pattern-matches `Success`/`Error` instead of try/catching.

---

## 7. DI graph (Koin)

`cmp-navigation/.../di/KoinModules.kt` aggregates **every** Koin module. There is no decentralized auto-discovery — the `allModules` list is the canonical wiring point:

```kotlin
val allModules = listOf(
    coreDataModule, domainModule, databaseModule, datastoreModule,
    networkModule, analyticsModule, syncModule, uiModule,
    dispatcherModule, homeModule, featureMoreModule, featureWealthModule,
    featureZakatModule, appModule,
)
```

Each module's responsibility:

| Module                | Owns                                                              |
| --------------------- | ----------------------------------------------------------------- |
| `commonModule`        | `DispatcherManager` (io/main/default dispatchers).                |
| `platformModule`      | Platform-specific singletons (Context, file paths, app config).   |
| `networkModule`       | Ktor `HttpClient`, Ktorfit factory, all `*ApiService` instances.  |
| `databaseModule`      | Room database + DAOs (singleton).                                 |
| `datastoreModule`     | Encrypted DataStore + repository for read/write of preference keys. |
| `coreDataModule`      | Repositories: `WealthRepository`, `ZakatRepository`, `MarketRepository`, `SettingsRepository`, `NetworkMonitor`. |
| `domainModule`        | Use cases (one per atomic operation — `GetAllWealthUseCase`, `SyncGoldPriceUseCase`, etc.). |
| `uiModule`            | Compose theming, app icons, MVI plumbing utilities.               |
| `analyticsModule`     | Firebase Analytics wrapper (platform-specific Android/iOS/Desktop variants). |
| `syncModule`          | `PriceSyncInteractor`, `PriceSyncScheduler` (platform-specific scheduler).      |
| `homeModule`          | `WealthListViewModel` (consumed by HomeScreen), `ChartsViewModel`. |
| `featureWealthModule` | `EditWealthViewModel` (`WealthListViewModel` is in `homeModule`). |
| `featureZakatModule`  | `ZakatListViewModel`.                                             |
| `featureMoreModule`   | `MoreViewModel`, `UserProfileViewModel`, `PreferencesViewModel`.  |
| `appModule`           | `AppViewModel`, `RootNavViewModel`, `AuthenticatedNavbarNavigationViewModel`. |

ViewModels are registered with `viewModelOf(::TheViewModel)` (Koin's reflection-based constructor injection). To inject one into a Composable, call `koinViewModel<TheViewModel>()`.

---

## 8. MVI pattern

Every screen in this app follows the same shape, codified in `cmpCore/ui/src/commonMain/kotlin/com/amosh/zakwa/core/ui/BaseViewModel.kt`:

```kotlin
abstract class BaseViewModel<S, E, A>(initialState: S) : ViewModel() {
    val stateFlow: StateFlow<S>            // UI snapshot
    val eventFlow: Flow<E>                 // one-shot effects (toast, navigate)
    fun trySendAction(action: A)           // input from the screen

    protected abstract fun handleAction(action: A)
}
```

The three type parameters are usually surfaced by a per-feature `*Contract` object with three sealed hierarchies:

```kotlin
object Contract {
    sealed class Event {                          // user input (A)
        data object OnFirstEvent : Event()
        data class OnEventWithArgs(val item: T) : Event()
    }
    data class ViewState(val state: State)         // UI state (S)
    sealed class Effect {                          // one-shot output (E)
        data class ShowError(val message: UiText) : Effect()
        data class ShowSuccess(val message: UiText) : Effect()
    }
}
```

The ViewModel observes use-case flows via `combine` and `onEach`, updates `mutableStateFlow` for UI changes, and pushes one-shot events via `sendEvent(...)`. The screen collects `stateFlow` with `collectAsStateWithLifecycle()` for the durable UI state and `eventFlow` separately for transient effects.

**Concrete example — `WealthListViewModel`** (after the Phase 1 cold-cache fix): combines five flows (wealth list, user data, exchange rate, gold price, silver price) into a `CombinedWealthData` and runs `performReactiveCalculations` on every emission. The two price flows are seeded with `onStart { emit(MetalPriceDto()) }` so the screen renders a placeholder + loading spinner immediately and re-renders the moment real prices arrive, without ever needing the app to restart. The `isGoldPriceLoading` / `isSilverPriceLoading` flags toggle once `gram24k > 0.0`.

When adding a new screen, mirror an existing feature exactly: `XxxContract`, `XxxViewModel : BaseViewModel<...>`, `XxxScreen(navController, vm = koinViewModel())`. Don't reinvent state holders.

---

## 9. Background sync

Three platforms, three scheduler implementations — but all of them call into the same `PriceSyncInteractor` defined in `cmpCore:sync`.

```
PriceSyncInteractor.run()
  └── SyncGoldPriceUseCase
  └── SyncSilverPriceUseCase
  └── SyncCurrencyExchangeRateUseCase
       (each delegates to MarketRepository.syncXxx() → firebaseApiService.getLatestData())
```

| Platform | Scheduler                  | Mechanism                                                                                |
| -------- | -------------------------- | ---------------------------------------------------------------------------------------- |
| Android  | `AndroidPriceSyncScheduler`| Enqueues `PriceSyncWorker` (CoroutineWorker) via WorkManager with retry-backoff.         |
| iOS      | `IosPriceSyncScheduler`    | Registers a `BGAppRefreshTask` for `com.amosh.zakwa.priceSync` in `AppDelegate`; system fires every ~6h. |
| Desktop  | `DesktopPriceSyncScheduler`| `kotlinx.coroutines.Timer`-style polling loop while the app is running.                  |

The `historySyncCursor` (a Unix-seconds timestamp in DataStore) guarantees the sync is incremental — each run starts from where the previous one left off, not from epoch.

Note: the `MarketRepositoryImpl.appConfigurationFlow` is `shareIn`-ed with `SharingStarted.WhileSubscribed(5000)`. As long as one subscriber is alive (the app's main scope keeps it alive on launch), the network sync runs and populates DataStore. This is why the Phase 1 cold-cache fix is needed: a one-shot `.first()` could grab a stale/empty snapshot before the shared flow's first real emission.

---

## 10. iOS entry point

`cmp-ios/iosApp/iOSApp.swift` is the Swift `@main` entry. Its responsibilities, in order:

1. **`AppDelegate.application(_:didFinishLaunchingWithOptions:)`**
   - Initializes Firebase (`FirebaseApp.configure()`).
   - Calls `KoinExtKt.doInitKoin()` — a Kotlin top-level function in `cmp-shared` that bootstraps the Koin DI graph with `KoinModules.allModules`.
   - Registers a `BGAppRefreshTask` handler for `"com.amosh.zakwa.priceSync"` and schedules the first run.
2. **`ContentView`** wraps a `UIViewControllerRepresentable` that calls `ViewControllerKt.viewController()` — another Kotlin top-level function in `cmp-shared` that returns the Compose `UIViewController`.
3. The framework is named `ComposeApp` and is produced by the `cmp-shared` Kotlin CocoaPods plugin (`isStatic = true`). Xcode consumes it via the Podfile at `cmp-ios/Podfile`.

When Kotlin code in `cmp-shared` changes (new public class, etc.), run `./gradlew :cmp-shared:podInstall` before reopening Xcode. App versioning flows from `version.properties` → `Config.xcconfig` via the `syncIosVersion` Gradle task, which runs automatically before any `*ios*` task.

---

## 11. Where to read more

- **Convention plugins** (how a new module should be wired) → `build-logic/convention/src/main/kotlin/` and the "Convention plugins" section of `CLAUDE.md`.
- **MVI canonical example** → `feature/zakat/src/commonMain/kotlin/com/amosh/zakwa/feature/zakat/screen/zakatList/`.
- **Backend code** → `functions/` (Node 22, independent of the Gradle build).
- **Module dependency report** → run `./gradlew createModuleGraph` to regenerate `docs/module_graph.dot`.

For ad-hoc spelunking, search by module name in your IDE (project structure → `Modules`). Type-safe accessors mean `projects.cmpCore.foo` will always jump to `cmpCore/foo/build.gradle.kts`.
