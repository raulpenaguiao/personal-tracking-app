# LifeLog — Frontend (Android)

Kotlin + Jetpack Compose Android app for the LifeLog personal logging system. Connects to your self-hosted VPS backend to record timestamped entries and display trends.

---

## Tech stack

| Component | Technology |
|---|---|
| Language | Kotlin |
| UI | Jetpack Compose |
| Navigation | Compose Navigation 2 |
| HTTP | Retrofit 2 + OkHttp 4 |
| Token storage | EncryptedSharedPreferences (Android Keystore) |
| Dependency injection | Hilt |
| Async | Kotlin Coroutines + Flow |
| Charts | Vico |
| Build | Gradle (Kotlin DSL) |
| Min SDK | 26 (Android 8.0) |
| Target SDK | 35 |

---

## Project structure

```
frontend/
├── build.gradle.kts
├── local.properties          ← set BASE_URL here (not committed)
├── app/
│   ├── src/main/
│   │   ├── AndroidManifest.xml
│   │   └── kotlin/com/yourname/lifelog/
│   │       ├── MainActivity.kt
│   │       ├── navigation/
│   │       │   └── AppNavGraph.kt        ← all routes defined here
│   │       ├── auth/
│   │       │   ├── LoginScreen.kt
│   │       │   ├── PinScreen.kt
│   │       │   └── AuthViewModel.kt
│   │       ├── dashboard/
│   │       │   └── DashboardScreen.kt    ← 6 category buttons
│   │       ├── category/
│   │       │   ├── health/
│   │       │   │   ├── HealthScreen.kt
│   │       │   │   └── HealthViewModel.kt
│   │       │   ├── mind/
│   │       │   ├── finances/
│   │       │   ├── projects/
│   │       │   ├── learning/
│   │       │   └── social/
│   │       ├── charts/
│   │       │   ├── ChartsScreen.kt       ← shared charts composable
│   │       │   └── ChartsViewModel.kt
│   │       ├── data/
│   │       │   ├── api/
│   │       │   │   ├── ApiService.kt     ← Retrofit interface
│   │       │   │   └── models/           ← request/response data classes
│   │       │   ├── repository/
│   │       │   │   ├── AuthRepository.kt
│   │       │   │   └── LogRepository.kt
│   │       │   └── local/
│   │       │       └── TokenStore.kt     ← Keystore-backed token storage
│   │       └── di/
│   │           └── AppModule.kt          ← Hilt module
│   └── src/test/
└── gradle/
```

---

## Configuration

Set your VPS base URL in `local.properties` (this file is gitignored):

```properties
BASE_URL=https://yourdomain.com/api/v1/
```

This is read in `build.gradle.kts` and injected as a `BuildConfig` field:

```kotlin
buildConfigField("String", "BASE_URL", "\"${localProperties["BASE_URL"]}\"")
```

---

## Getting started

### Prerequisites

- Android Studio Hedgehog (2023.1.1) or later
- JDK 17
- An Android device or emulator running Android 8.0+

### Steps

1. Open `frontend/` as a project in Android Studio
2. Set `BASE_URL` in `local.properties`
3. Let Gradle sync
4. Run on a device or emulator (`Shift+F10`)

---

## Authentication flow

### First launch (new device)

1. `LoginScreen` — user enters username and password
2. App calls `POST /auth/login`, receives a short-lived pre-token
3. `PinScreen` — user enters their PIN
4. App calls `POST /auth/pin` with the pre-token, PIN, and device fingerprint
5. Server returns a JWT; app stores it in `EncryptedSharedPreferences`

### Returning recognised device

1. App reads the stored token; if expired, navigates to `PinScreen` only
2. User enters PIN → new JWT issued → stored again

### Token injection

An OkHttp interceptor reads the token from `TokenStore` and adds it to every outgoing request:

```kotlin
class AuthInterceptor @Inject constructor(
    private val tokenStore: TokenStore
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenStore.getToken() ?: return chain.proceed(chain.request())
        val request = chain.request().newBuilder()
            .addHeader("Authorization", "Bearer $token")
            .build()
        return chain.proceed(request)
    }
}
```

A 401 response clears the stored token and redirects to `LoginScreen`.

---

## Screen map

```
LoginScreen
    └── PinScreen
            └── DashboardScreen
                    ├── HealthScreen
                    │       ├── Log weight
                    │       ├── Log steps
                    │       ├── Log sleep
                    │       └── Charts tab
                    ├── MindScreen
                    │       ├── Journal entry
                    │       ├── Mood rating
                    │       └── Charts tab
                    ├── FinancesScreen
                    │       ├── Log expense
                    │       ├── Log income
                    │       └── Charts tab
                    ├── ProjectsScreen
                    │       ├── Log update
                    │       ├── Log time
                    │       └── Charts tab
                    ├── LearningScreen
                    │       ├── Book progress
                    │       ├── Course note
                    │       └── Charts tab
                    └── SocialScreen
                            ├── Log interaction
                            ├── Log event
                            └── Charts tab
```

---

## Logging a new entry

Every category screen uses the same `LogRepository.postEntry()` call:

```kotlin
data class LogRequest(
    val category: String,
    val action: String,
    val value: String,
    val note: String? = null
)

// In the ViewModel:
viewModelScope.launch {
    val result = logRepository.postEntry(
        LogRequest(category = "health", action = "weight", value = "82.3")
    )
    result.onSuccess { showSnackbar("Logged!") }
          .onFailure { showError(it.message) }
}
```

---

## Charts

The `ChartsScreen` calls `GET /api/v1/stats/{category}/{action}?days=30` and renders the time-series with Vico's `CartesianChartHost`. Each category screen includes a bottom tab that navigates to the chart for that category.

---

## Dependencies (key)

```kotlin
// build.gradle.kts (app)
implementation("com.squareup.retrofit2:retrofit:2.9.0")
implementation("com.squareup.retrofit2:converter-gson:2.9.0")
implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
implementation("androidx.security:security-crypto:1.1.0-alpha06")
implementation("com.patrykandpatrick.vico:compose:1.13.0")
implementation("androidx.hilt:hilt-navigation-compose:1.1.0")
implementation("com.google.dagger:hilt-android:2.50")
```

---

## Build variants

| Variant | BASE_URL | Logging |
|---|---|---|
| `debug` | from `local.properties` | OkHttp logging enabled |
| `release` | from CI secret / `local.properties` | No HTTP logging |

Build a release APK:

```bash
./gradlew assembleRelease
```

Sign with your keystore before distribution.

---

## Running tests

```bash
./gradlew test              # unit tests
./gradlew connectedCheck    # instrumented tests (requires device/emulator)
```
