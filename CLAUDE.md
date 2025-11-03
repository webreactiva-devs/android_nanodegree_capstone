# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ultimate Scrobbler is an Android app that scrobbles played songs from Spotify to Last.fm. The app listens for Spotify broadcasts, tracks song playback, and sends scrobble data to Last.fm using their API.

## Prerequisites

### Required Configuration Variables

The following variables must be set in `gradle.properties` for the project to build:

```gradle
CAPSTONE_KEYSTORE_PASSWORD
CAPSTONE_KEY_PASSWORD
LAST_FM_API_KEY
LAST_FM_API_SECRET
```

Note: The keystore file is `capstone.jks` at the project root.

## Build Commands

```bash
# Build the project
./gradlew build

# Build debug variant
./gradlew assembleDebug

# Build release variant
./gradlew assembleRelease

# Install debug build on device
./gradlew installDebug

# Clean build
./gradlew clean

# Check for dependency updates
./gradlew dependencyUpdates

# View method count and dex information
./gradlew countDebugDexMethods
```

## Architecture

This project follows **Clean Architecture** with a clear separation of concerns across multiple modules:

### Module Structure

- **app**: Android UI layer (Activities, Fragments, ViewModels, Services, BroadcastReceivers, Widgets)
- **domain**: Business logic layer (Use Cases, Domain Models, Repository interfaces)
- **data**: Data layer implementation (Repository implementations, Entity models, Mappers)
- **remote**: Remote data source (Last.fm API integration via Retrofit)
- **cache**: Local data source (SQLite database via Schematic, SharedPreferences via RxPreferences)

### Data Flow

1. **UI Layer** → ViewModels subscribe to Use Cases
2. **Domain Layer** → Use Cases orchestrate business logic and call Repository interfaces
3. **Data Layer** → Repositories coordinate between Remote and Cache sources
4. **Remote/Cache** → Actual data fetching/storage implementation

### Key Architectural Patterns

- **Dependency Injection**: Dagger 2 with `ApplicationComponent` as the root component
  - Modules: `ApplicationModule`, `ActivityBindingModule`, `FragmentBindingModule`, `ServiceBindingModule`, `NetworkModule`
  - Injection targets: Activities, Fragments, Services

- **Reactive Programming**: RxJava2 for asynchronous operations
  - Use Cases extend `ObservableUseCase`, `SingleUseCase`, or `CompletableUseCase`
  - RxRelay for event streams (song playback events)
  - RxBinding for UI events

- **MVVM Pattern**: ViewModels manage UI state with Android Architecture Components
  - ViewStates represent UI configuration
  - ViewModelFactory for ViewModel creation with dependencies

- **Repository Pattern**: Abstract data sources behind repository interfaces in the domain layer

### Core Application Flow

1. **Application Start**: `UltimateScrobblerApplication.onCreate()` initializes Dagger and starts `ScrobblerService` as foreground service
2. **Song Detection**: `SpotifyReceiver` listens for Spotify metadata broadcast intents
3. **Scrobble Logic**: Songs are debounced (10s) and queued for scrobbling after 50% playback
4. **API Communication**: Firebase JobDispatcher schedules `ScrobblePlayedSongsService` and `SendNowPlayingService` for batch uploads to Last.fm
5. **Persistence**: Songs are stored in SQLite (via Schematic) and uploaded in batches

### Build Variants

- **debug**: Includes development tools (Stetho, Chuck, LeakCanary, OkLog)
  - Different `CustomApplication` and `NetworkModule` implementations
  - Suffix: `.debug`
- **release**: Production build with release signing

### Version Management

Version is managed via `gradle.properties`:
- `VERSION_MAJOR`, `VERSION_MINOR`, `VERSION_PATCH`
- Debug builds append git commit hash to version name
- Version code calculated as: `major * 1_000_000 + minor * 1_000 + patch`

### Database

Database schema is code-generated using **Schematic** annotation processor:
- `SongsDatabase` defines the database
- `PlayedSongColumns` and `InfoSongColumns` define tables
- `SongsProvider` is the auto-generated ContentProvider

### Key Third-Party Integrations

- **Last.fm API**: REST API integration via Retrofit + RxJava2 adapters
- **Spotify**: Broadcast receiver integration (no official SDK)
- **Firebase**: JobDispatcher for background tasks, Crashlytics for crash reporting, Performance monitoring
- **ButterKnife**: View binding (note: now deprecated but used in this legacy project)
- **ThreeTen BP**: Java 8 Time API backport for date/time handling

## Testing

No test task was found in the Gradle configuration. To add tests, implement them in the standard Android test directories and run with:
```bash
./gradlew test          # Unit tests
./gradlew connectedAndroidTest  # Instrumented tests
```
