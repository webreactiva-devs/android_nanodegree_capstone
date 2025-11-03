# Ultimate Scrobbler - Architecture Documentation

## Table of Contents
1. [Overview](#overview)
2. [Module Structure](#module-structure)
3. [Dependency Injection](#dependency-injection)
4. [Data Flow Architecture](#data-flow-architecture)
5. [Scrobbling Workflow](#scrobbling-workflow)
6. [Key Patterns and Components](#key-patterns-and-components)
7. [Package Structure](#package-structure)

---

## Overview

Ultimate Scrobbler is an Android application that automatically scrobbles (tracks) songs played on Spotify to Last.fm. The application follows **Clean Architecture** principles with a clear separation of concerns across five independent modules.

### Technology Stack

- **Language**: Java 8
- **Architecture**: Clean Architecture (multi-module)
- **Dependency Injection**: Dagger 2 (v2.14.1)
- **Reactive Programming**: RxJava2 (v2.1.8) + RxAndroid + RxBinding + RxRelay
- **UI Pattern**: MVVM with Android Architecture Components
- **Networking**: Retrofit 2.3.0 with Moshi converter and RxJava2 adapter
- **Local Storage**: SQLite via Schematic (ContentProvider generation) + RxPreferences
- **Background Jobs**: Firebase JobDispatcher (v0.8.5)
- **Date/Time**: ThreeTen BP/ABP (Java 8 Time API backport)
- **View Binding**: ButterKnife (v8.8.1) - legacy
- **Immutability**: AutoValue (v1.5.3) for value objects
- **Crash Reporting**: Firebase Crashlytics
- **Performance Monitoring**: Firebase Performance

### Core Principles

1. **Separation of Concerns**: Each module has a single, well-defined responsibility
2. **Dependency Rule**: Dependencies point inward - outer layers depend on inner layers, never the reverse
3. **Abstraction**: Business logic depends on interfaces, not concrete implementations
4. **Testability**: Pure business logic in domain module with no Android dependencies
5. **Reactive Streams**: Asynchronous operations handled via RxJava observables

---

## Module Structure

The application consists of five Gradle modules organized in a layered architecture:

```
┌─────────────────────────────────────────────────────┐
│                      app                            │
│  (UI Layer: Activities, Fragments, ViewModels,     │
│   Services, BroadcastReceivers, Widgets)            │
└─────────────┬───────────────────────────────────────┘
              │ depends on
              ↓
┌─────────────────────────────────────────────────────┐
│                    domain                           │
│  (Business Logic: Use Cases, Domain Models,         │
│   Repository Interfaces)                            │
│  Pure Java - No Android Dependencies               │
└─────────────┬───────────────────────────────────────┘
              │ depends on
              ↓
┌─────────────────────────────────────────────────────┐
│                     data                            │
│  (Repository Implementations, Entity Models,        │
│   Data Source Interfaces, Mappers)                  │
└─────────────┬───────────────────────────────────────┘
              │ depends on
         ┌────┴────┐
         ↓         ↓
    ┌────────┐  ┌────────┐
    │ remote │  │ cache  │
    │ (API)  │  │ (DB)   │
    └────────┘  └────────┘
```

### 1. App Module

**Type**: Android Application Module
**Location**: `/app`

**Responsibilities**:
- Android UI layer (Activities, Fragments)
- ViewModels and ViewStates (MVVM pattern)
- Android Services (foreground service, job services)
- BroadcastReceivers (Spotify integration)
- Dependency Injection setup (Dagger components and modules)
- Application-level configuration

**Dependencies**:
- `domain` - Business logic layer
- `data` - Data layer implementation
- `remote` - Last.fm API integration
- `cache` - Local persistence

**Key Libraries**:
- Android Support Library (v27.0.2): AppCompat, Design, RecyclerView, CardView, Palette
- Architecture Components: Lifecycle Extensions (v1.0.0)
- Dagger 2 (v2.14.1) with Android support
- RxJava2 ecosystem (RxJava, RxAndroid, RxBinding, RxRelay)
- Firebase: JobDispatcher, Crashlytics, Performance
- ButterKnife (v8.8.1) for view binding
- UI libraries: IndicatorSeekBar, ChipCloud, Picasso, FlexBox

**Build Variants**:

- **Debug**:
  - Application ID suffix: `.debug`
  - Version name suffix: `-debug-{git-hash}`
  - Debug tools: Stetho, Chuck, OkLog, LeakCanary
  - Custom `NetworkModule` with HTTP interceptors

- **Release**:
  - Production build
  - Signed with release keystore
  - ProGuard disabled (can be enabled)

**Version Management** (`gradle.properties`):
```properties
VERSION_MAJOR=1
VERSION_MINOR=0
VERSION_PATCH=0

versionCode = major * 1,000,000 + minor * 1,000 + patch
versionName = "major.minor.patch"
```

### 2. Domain Module

**Type**: Pure Java Module
**Location**: `/domain`

**Responsibilities**:
- Business logic implementation via Use Cases
- Domain model definitions (pure business entities)
- Repository interface definitions
- Business rule validation
- No framework dependencies

**Dependencies**: None (pure Java)

**Key Libraries**:
- RxJava2 (v2.1.8) - for reactive streams
- RxRelay (v2.0.0) - for event buses
- ThreeTen BP (v1.3.6) - date/time handling
- AutoValue (v1.5.3) - immutable value objects
- RxSealedUnions (v1.0.0) - discriminated unions
- Javax Inject (v1) - JSR-330 annotations

**Core Components**:
- **Use Cases**: Encapsulate single business operations (see [Data Flow Architecture](#data-flow-architecture))
- **Domain Models**: `PlayedSong`, `ScrobbledSong`, `InfoSong`, `Credentials`, `UserConfiguration`
- **Repository Interfaces**: `ScrobblerRepository`, `ConfigurationRepository`
- **Errors**: `InvalidCredentialsError`

**Design Philosophy**:
> The domain module is the heart of the application. It contains zero Android dependencies, making it easily testable and portable. All business rules are enforced here.

### 3. Data Module

**Type**: Pure Java Module
**Location**: `/data`

**Responsibilities**:
- Implementation of repository interfaces from domain layer
- Coordination between remote and cache data sources
- Entity model definitions (data layer representations)
- Mapping between domain models and entities
- Data source abstraction interfaces

**Dependencies**: `domain`

**Key Libraries**:
- RxJava2, ThreeTen BP
- AutoValue
- Javax Inject

**Core Components**:
- **Repository Implementations**:
  - `ScrobblerDataRepository` (implements `ScrobblerRepository`)
  - `UserConfigurationDataRepository` (implements `ConfigurationRepository`)
- **Data Source Interfaces**:
  - `ScrobblerRemote` - remote data source contract
  - `SongsCache` - local songs data source contract
  - `ConfigurationCache` - preferences data source contract
- **Entity Models**: `PlayedSongEntity`, `ScrobbledSongEntity`, `InfoSongEntity`, `CredentialsEntity`
- **Mappers**: Bidirectional conversion between domain models and entities

**Repository Pattern**:
```java
@Inject
public ScrobblerDataRepository(
    Provider<ScrobblerRemote> scrobblerRemote,  // Lazy injection
    ConfigurationCache configurationCache,
    SongsCache songsCache,
    // Mappers for data transformation
    PlayedSongMapper playedSongMapper,
    ScrobbledSongMapper scrobbledSongMapper,
    InfoSongMapper infoSongMapper)
```

The repository coordinates between remote and cache sources, deciding when to fetch from network vs. local storage, and handles data synchronization.

### 4. Remote Module

**Type**: Android Library Module
**Location**: `/remote`

**Responsibilities**:
- Last.fm API integration via Retrofit
- Network request/response models
- API authentication and signature generation
- Mapping between remote models and data entities

**Dependencies**: `data`

**Key Libraries**:
- Retrofit (v2.3.0) with Moshi converter and RxJava2 call adapter
- Moshi Lazy Adapters (v2.1) - JSON unwrapping
- AutoValue with Moshi extensions (v0.4.5)
- Dagger 2 (for injection in debug tools)
- Timber (v4.5.1) - logging

**API Configuration**:
- **Base URL**: `https://ws.audioscrobbler.com/2.0`
- **Format**: JSON
- **Authentication**: MD5 signature based on sorted parameters + API secret
- **Response Format**: Moshi with `@Wrapped` annotation for JSON path unwrapping

**Retrofit Service Interface** (`ScrobblerService.java`):
```java
@FormUrlEncoded
@POST(WS_PATH)
@Wrapped(path = {"session", "key"})
Single<String> requestMobileSessionToken(@FieldMap Map<String, String> parameters);

@FormUrlEncoded
@POST(WS_PATH)
@Wrapped(path = {"scrobbles", "scrobble"})
Observable<List<ScrobbledSongModel>> scrobbleMultiple(@FieldMap Map<String, String> parameters);

@FormUrlEncoded
@POST(WS_PATH)
@Wrapped(path = {"track"})
Single<InfoSongModel> requestSongInformation(@FieldMap Map<String, String> parameters);
```

**API Signature Generation**:
```java
// Sort all parameters alphabetically, concatenate key+value pairs, append secret
// Then compute MD5 hash
private String getSignature(SortedMap<String, String> params) {
    StringBuilder signatureBuilder = new StringBuilder();
    for (Map.Entry<String, String> entry : params.entrySet()) {
        signatureBuilder.append(entry.getKey()).append(entry.getValue());
    }
    signatureBuilder.append(apiSecret);
    return ByteString.encodeUtf8(signatureBuilder.toString()).md5().hex();
}
```

### 5. Cache Module

**Type**: Android Library Module
**Location**: `/cache`

**Responsibilities**:
- Local data persistence (SQLite database)
- User preferences storage (SharedPreferences)
- Database schema definition via Schematic annotations
- Mapping between database cursors/ContentValues and data entities

**Dependencies**: `data`

**Key Libraries**:
- Schematic (v0.7.0) - ContentProvider code generation
- RxPreferences (v2.0.0-RC3) - reactive SharedPreferences wrapper
- ThreeTen BP, Timber

**Database Schema** (Schematic):

```java
@Database(version = 1, packageName = "com.github.niltsiar.ultimatescrobbler.cache.provider")
public class SongsDatabase {
    @Table(PlayedSongColumns.class)
    public static final String PLAYED_SONGS = "played_songs";

    @Table(InfoSongColumns.class)
    public static final String INFO_SONG = "info_song";

    @Table(PlayedSongColumns.class)
    public static final String CURRENT_SONG = "current_song";
}
```

**Table Definitions**:

**PlayedSongColumns**:
```java
@DataType(TEXT) @PrimaryKey(onConflict = REPLACE) String ID = "_id";
@DataType(TEXT) @NotNull String TRACK_NAME = "track_name";
@DataType(TEXT) @NotNull String ARTIST_NAME = "artist_name";
@DataType(TEXT) @NotNull String ALBUM_NAME = "album_name";
@DataType(INTEGER) @NotNull String LENGTH = "length";
@DataType(INTEGER) @NotNull String TIMESTAMP = "played_instant";
@DataType(INTEGER) @NotNull String SCROBBLED = "scrobbled";
```

**InfoSongColumns**:
```java
@DataType(INTEGER) @PrimaryKey(onConflict = REPLACE) String ID = "_id";
@DataType(TEXT) @NotNull String TRACK_NAME, ARTIST_NAME, ALBUM_NAME;
@DataType(TEXT) @NotNull String ALBUM_ARTIST_NAME, ALBUM_ART_URL;
@DataType(TEXT) @NotNull String TAGS, WIKI_CONTENT;
```

**Auto-Generated**: `SongsProvider` ContentProvider is generated at compile-time by Schematic annotation processor.

**RxPreferences Pattern**:
```java
@Inject
public ConfigurationCacheImpl(RxSharedPreferences preferences) {
    mobileSessionPreference = preferences.getString(MOBILE_SESSION_PREFERENCE, "");
    usernamePreference = preferences.getString(USERNAME_PREFERENCE, "");
    numberOfSongsPerBatchPreference = preferences.getInteger(
        NUMBER_OF_SONGS_PER_BATCH_PREFERENCE, 10);
}

@Override
public Consumer<? super String> saveMobileSessionToken() {
    return mobileSessionPreference.asConsumer();  // Returns RxJava Consumer
}

@Override
public Single<String> getMobileSessionToken() {
    return mobileSessionPreference.asObservable().firstOrError();
}
```

---

## Dependency Injection

The application uses **Dagger 2** (v2.14.1) with the Android support library (`dagger.android`) for dependency injection. The DI graph is configured at application startup and provides dependencies throughout the app lifecycle.

### Component Hierarchy

```
ApplicationComponent (Singleton Scope)
    │
    ├── ActivityBindingModule
    │   ├── ConfigurationActivity
    │   └── SongDetailsActivity
    │
    ├── FragmentBindingModule
    │   ├── PlayedSongsFragment
    │   └── ScrobbledSongsFragment
    │
    └── ServiceBindingModule
        ├── ScrobblerService
        ├── SendNowPlayingService
        └── ScrobblePlayedSongsService
```

### ApplicationComponent

**Location**: `/app/src/main/java/com/github/niltsiar/ultimatescrobbler/di/ApplicationComponent.java`

```java
@Singleton
@Component(modules = {
    ApplicationModule.class,
    ActivityBindingModule.class,
    ServiceBindingModule.class,
    FragmentBindingModule.class,
    AndroidSupportInjectionModule.class,  // Dagger Android support
    NetworkModule.class
})
public interface ApplicationComponent extends AndroidInjector<UltimateScrobblerApplication> {

    @Component.Builder
    interface Builder {
        @BindsInstance
        Builder application(Application application);
        ApplicationComponent build();
    }
}
```

**Initialization** (`UltimateScrobblerApplication.onCreate()`):
```java
DaggerApplicationComponent.builder()
    .application(this)
    .build()
    .inject(this);
```

### Dagger Modules

#### 1. ApplicationModule

**Location**: `/app/src/main/java/com/github/niltsiar/ultimatescrobbler/di/module/ApplicationModule.java`

**Provides**:

**Core Dependencies**:
```java
@Provides
@Singleton
Context provideContext(Application application) {
    return application;
}
```

**API Credentials** (from BuildConfig):
```java
@Provides
@Singleton
@ApiKey
String provideApiKey() {
    return BuildConfig.LAST_FM_API_KEY;
}

@Provides
@Singleton
@ApiSecret
String provideApiSecret() {
    return BuildConfig.LAST_FM_API_SECRET;
}

@Provides
@Singleton
@MobileSessionToken
Provider<String> provideMobileSessionToken(ConfigurationRepository configurationRepository) {
    // Lazy Provider - fetches token when needed
}
```

**Repositories** (interface → implementation binding):
```java
@Binds
@Singleton
ScrobblerRepository bindScrobblerRepository(ScrobblerDataRepository scrobblerDataRepository);

@Binds
@Singleton
ConfigurationRepository bindConfigurationRepository(
    UserConfigurationDataRepository userConfigurationDataRepository);
```

**Data Sources**:
```java
@Binds
@Singleton
ScrobblerRemote bindScrobblerRemote(ScrobblerRemoteImpl scrobblerRemoteImpl);

@Binds
@Singleton
ConfigurationCache bindConfigurationCache(ConfigurationCacheImpl configurationCacheImpl);

@Binds
@Singleton
SongsCache bindSongsCache(SongsCacheImpl songsCacheImpl);
```

**Network Layer**:
```java
@Provides
@Singleton
ScrobblerService provideScrobblerService(OkHttpClient client) {
    return new Retrofit.Builder()
        .baseUrl("https://ws.audioscrobbler.com/2.0/")
        .addConverterFactory(MoshiConverterFactory.create(moshi))
        .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
        .client(client)
        .build()
        .create(ScrobblerService.class);
}
```

**Background Jobs**:
```java
@Provides
@Singleton
FirebaseJobDispatcher provideJobDispatcher(Application application) {
    return new FirebaseJobDispatcher(new GooglePlayDriver(application));
}
```

**SharedPreferences**:
```java
@Provides
@Singleton
RxSharedPreferences provideRxSharedPreferences(Application application) {
    SharedPreferences preferences = PreferenceManager.getDefaultSharedPreferences(application);
    return RxSharedPreferences.create(preferences);
}
```

**Use Cases** (with RxJava schedulers):
```java
@Provides
@Singleton
GetPlayedSongsUseCase provideGetPlayedSongsUseCase(ScrobblerRepository repository) {
    return new GetPlayedSongsUseCase(
        repository,
        Schedulers.io(),              // Execute on I/O thread
        AndroidSchedulers.mainThread() // Post results on main thread
    );
}

// ... 12 more Use Case providers (13 total)
```

#### 2. NetworkModule

**Debug Variant** (`/app/src/debug/java/.../NetworkModule.java`):
```java
@Module
public class NetworkModule {
    @Provides
    @Singleton
    OkHttpClient provideOkHttpClient(Context context) {
        return new OkHttpClient.Builder()
            .addInterceptor(new HttpLoggingInterceptor().setLevel(BODY))
            .addInterceptor(new OkLogInterceptor())
            .addInterceptor(new ChuckInterceptor(context))  // Network inspector
            .addNetworkInterceptor(new StethoInterceptor())  // Chrome DevTools
            .build();
    }
}
```

**Release Variant** (`/app/src/release/java/.../NetworkModule.java`):
```java
@Module
public class NetworkModule {
    @Provides
    @Singleton
    OkHttpClient provideOkHttpClient() {
        return new OkHttpClient.Builder().build();  // No interceptors
    }
}
```

#### 3. ActivityBindingModule

**Location**: `/app/src/main/java/com/github/niltsiar/ultimatescrobbler/di/module/ActivityBindingModule.java`

```java
@Module
public abstract class ActivityBindingModule {

    @ContributesAndroidInjector
    abstract ConfigurationActivity contributeConfigurationActivityInjector();

    @ContributesAndroidInjector
    abstract SongDetailsActivity contributeSongDetailsActivityInjector();
}
```

**Usage in Activity**:
```java
public class ConfigurationActivity extends AppCompatActivity {
    @Inject ConfigurationViewModelFactory viewModelFactory;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        AndroidInjection.inject(this);  // Dagger injection
        super.onCreate(savedInstanceState);

        ConfigurationViewModel viewModel = ViewModelProviders.of(
            this, viewModelFactory).get(ConfigurationViewModel.class);
    }
}
```

#### 4. FragmentBindingModule

**Location**: `/app/src/main/java/com/github/niltsiar/ultimatescrobbler/di/module/FragmentBindingModule.java`

```java
@Module
public abstract class FragmentBindingModule {

    @ContributesAndroidInjector
    abstract PlayedSongsFragment contributePlayedSongsFragment();

    @ContributesAndroidInjector
    abstract ScrobbledSongsFragment contributeScrobbledSongsFragment();
}
```

#### 5. ServiceBindingModule

**Location**: `/app/src/main/java/com/github/niltsiar/ultimatescrobbler/di/module/ServiceBindingModule.java`

```java
@Module
public abstract class ServiceBindingModule {

    @ContributesAndroidInjector
    abstract ScrobblerService contributeScrobblerService();

    @ContributesAndroidInjector
    abstract SendNowPlayingService contributeSendNowPlayingServiceInjector();

    @ContributesAndroidInjector
    abstract ScrobblePlayedSongsService contributeScrobblePlayedSongsService();
}
```

**Usage in Service**:
```java
public class ScrobblerService extends Service implements HasServiceInjector {
    @Inject DispatchingAndroidInjector<Service> serviceDispatchingAndroidInjector;
    @Inject SpotifyReceiver spotifyReceiver;
    @Inject SavePlayedSongUseCase savePlayedSongUseCase;

    @Override
    public void onCreate() {
        AndroidInjection.inject(this);  // Dagger injection
        super.onCreate();
    }

    @Override
    public AndroidInjector<Service> serviceInjector() {
        return serviceDispatchingAndroidInjector;
    }
}
```

### Injection Flow

```
1. Application starts
   ↓
2. ApplicationComponent built in UltimateScrobblerApplication.onCreate()
   ↓
3. Activity/Fragment/Service created
   ↓
4. AndroidInjection.inject(this) called in onCreate()
   ↓
5. Dagger looks up subcomponent in binding module
   ↓
6. Dependencies injected into @Inject annotated fields
   ↓
7. Component ready to use
```

---

## Data Flow Architecture

The application follows a **unidirectional data flow** pattern typical of Clean Architecture, with clear boundaries between layers.

### Architecture Layers

```
┌────────────────────────────────────────────────────────────────┐
│                       Presentation Layer                       │
│  Activities, Fragments, ViewModels, ViewStates, Adapters       │
│                                                                 │
│  Responsibility: Display data, handle user interaction         │
└───────────────────────────┬────────────────────────────────────┘
                            │ calls Use Cases
                            │ observes ViewState
                            ↓
┌────────────────────────────────────────────────────────────────┐
│                         Domain Layer                           │
│  Use Cases, Domain Models, Repository Interfaces               │
│                                                                 │
│  Responsibility: Business logic, business rules                │
└───────────────────────────┬────────────────────────────────────┘
                            │ calls Repository interface
                            │ operates on Domain Models
                            ↓
┌────────────────────────────────────────────────────────────────┐
│                          Data Layer                            │
│  Repository Implementations, Entity Models, Mappers            │
│                                                                 │
│  Responsibility: Data coordination, entity mapping             │
└───────────────────────────┬────────────────────────────────────┘
                            │ coordinates sources
                            │ maps Entities ↔ Domain Models
                     ┌──────┴──────┐
                     ↓             ↓
         ┌──────────────────┐  ┌──────────────────┐
         │   Remote Layer   │  │   Cache Layer    │
         │  (Last.fm API)   │  │  (SQLite + Prefs)│
         │                  │  │                  │
         │  Responsibility: │  │  Responsibility: │
         │  Network calls   │  │  Local storage   │
         └──────────────────┘  └──────────────────┘
```

### Layer Interaction Pattern

#### 1. ViewModel → Use Case

**Example**: `ConfigurationViewModel`
**Location**: `/app/src/main/java/com/github/niltsiar/ultimatescrobbler/ui/configuration/ConfigurationViewModel.java`

```java
public class ConfigurationViewModel extends ViewModel {

    @Inject RetrieveUserConfigurationUseCase retrieveUserConfigurationUseCase;
    @Inject SaveUserConfigurationUseCase saveUserConfigurationUseCase;
    @Inject RequestMobileSessionTokenUseCase requestMobileSessionTokenUseCase;

    private BehaviorRelay<ConfigurationViewState> configurationViewStateBehaviorRelay;
    private CompositeDisposable disposables = new CompositeDisposable();

    public void loadConfiguration() {
        disposables.add(
            retrieveUserConfigurationUseCase.execute(null)  // Call Use Case
                .subscribe(
                    userConfiguration -> {
                        // Map domain model to ViewState
                        ConfigurationViewState viewState =
                            ConfigurationViewStateMapper.mapFromUserConfiguration(userConfiguration);
                        // Publish to UI
                        configurationViewStateBehaviorRelay.accept(viewState);
                    },
                    Timber::e
                )
        );
    }

    public void saveConfiguration(UserConfiguration config) {
        disposables.add(
            saveUserConfigurationUseCase.execute(config)
                .subscribe(
                    () -> { /* Success */ },
                    Timber::e
                )
        );
    }

    public Observable<ConfigurationViewState> getConfigurationViewState() {
        return configurationViewStateBehaviorRelay;  // UI subscribes to this
    }

    @Override
    protected void onCleared() {
        disposables.clear();  // Prevent memory leaks
        super.onCleared();
    }
}
```

**ViewState Pattern** (immutable):
```java
@AutoValue
public abstract class ConfigurationViewState {
    public abstract String getUsername();
    public abstract String getPassword();
    public abstract int getNumberOfSongsPerBatch();
    public abstract boolean isSendNowPlaying();
    public abstract boolean isInvalidCredentialsError();

    // Immutable updates via builder
    public ConfigurationViewState withUsername(String newUsername) {
        return toBuilder().setUsername(newUsername).build();
    }

    @AutoValue.Builder
    public abstract static class Builder {
        public abstract Builder setUsername(String username);
        public abstract Builder setPassword(String password);
        // ... more setters
        public abstract ConfigurationViewState build();
    }
}
```

**Activity observes ViewModel**:
```java
public class ConfigurationActivity extends AppCompatActivity {
    @Inject ConfigurationViewModelFactory viewModelFactory;
    private ConfigurationViewModel viewModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        AndroidInjection.inject(this);
        super.onCreate(savedInstanceState);

        viewModel = ViewModelProviders.of(this, viewModelFactory)
            .get(ConfigurationViewModel.class);

        // Subscribe to ViewState changes
        viewModel.getConfigurationViewState()
            .subscribe(this::renderViewState, Timber::e);

        viewModel.loadConfiguration();  // Trigger data load
    }

    private void renderViewState(ConfigurationViewState viewState) {
        // Update UI based on ViewState
        usernameEditText.setText(viewState.getUsername());
        passwordEditText.setText(viewState.getPassword());
        // ... more UI updates
    }
}
```

#### 2. Use Case Base Classes

**Location**: `/domain/src/main/java/com/github/niltsiar/ultimatescrobbler/domain/interactor/`

##### ObservableUseCase (for streams with multiple emissions)

```java
public abstract class ObservableUseCase<T, V> {

    private final Scheduler executionScheduler;    // Background thread
    private final Scheduler postExecutionScheduler; // Main thread

    protected ObservableUseCase(Scheduler executionScheduler,
                                Scheduler postExecutionScheduler) {
        this.executionScheduler = executionScheduler;
        this.postExecutionScheduler = postExecutionScheduler;
    }

    protected abstract Observable<T> buildUseCaseObservable(@Nullable V param);

    public Observable<T> execute(@Nullable V param) {
        return buildUseCaseObservable(param)
            .subscribeOn(executionScheduler)      // Execute on I/O thread
            .observeOn(postExecutionScheduler);   // Observe on main thread
    }
}
```

##### SingleUseCase (for one-shot operations)

```java
public abstract class SingleUseCase<T, V> {

    private final Scheduler executionScheduler;
    private final Scheduler postExecutionScheduler;

    protected SingleUseCase(Scheduler executionScheduler,
                           Scheduler postExecutionScheduler) {
        this.executionScheduler = executionScheduler;
        this.postExecutionScheduler = postExecutionScheduler;
    }

    protected abstract Single<T> buildUseCaseObservable(@Nullable V param);

    public Single<T> execute(@Nullable V param) {
        return buildUseCaseObservable(param)
            .subscribeOn(executionScheduler)
            .observeOn(postExecutionScheduler);
    }
}
```

##### CompletableUseCase (for side effects with no return value)

```java
public abstract class CompletableUseCase<T> {

    private final Scheduler executionScheduler;
    private final Scheduler postExecutionScheduler;

    protected CompletableUseCase(Scheduler executionScheduler,
                                Scheduler postExecutionScheduler) {
        this.executionScheduler = executionScheduler;
        this.postExecutionScheduler = postExecutionScheduler;
    }

    protected abstract Completable buildUseCaseObservable(@Nullable T param);

    public Completable execute(@Nullable T param) {
        return buildUseCaseObservable(param)
            .subscribeOn(executionScheduler)
            .observeOn(postExecutionScheduler);
    }
}
```

**Example Use Case Implementation**:
**GetPlayedSongsUseCase** (domain layer)
**Location**: `/domain/src/main/java/com/github/niltsiar/ultimatescrobbler/domain/interactor/playedsong/GetPlayedSongsUseCase.java`

```java
public class GetPlayedSongsUseCase extends SingleUseCase<List<PlayedSong>, Void> {

    private final ScrobblerRepository scrobblerRepository;

    @Inject
    public GetPlayedSongsUseCase(
        ScrobblerRepository scrobblerRepository,
        Scheduler executionScheduler,
        Scheduler postExecutionScheduler) {

        super(executionScheduler, postExecutionScheduler);
        this.scrobblerRepository = scrobblerRepository;
    }

    @Override
    protected Single<List<PlayedSong>> buildUseCaseObservable(@Nullable Void param) {
        return scrobblerRepository.getStoredPlayedSongs();  // Delegate to repository
    }
}
```

#### 3. Repository Interfaces (Domain Layer)

**ScrobblerRepository**
**Location**: `/domain/src/main/java/com/github/niltsiar/ultimatescrobbler/domain/repository/ScrobblerRepository.java`

```java
public interface ScrobblerRepository {

    // Authentication
    Single<String> requestMobileSessionToken(Credentials credentials);

    // Now Playing
    Completable sendNowPlaying(PlayedSong playedSong);
    Completable saveCurrentSong(PlayedSong playedSong);
    Single<PlayedSong> getCurrentSong(String songId);

    // Played Songs (local storage)
    Completable savePlayedSong(PlayedSong playedSong);
    Single<Long> countStoredPlayedSongs();
    Single<PlayedSong> getStoredPlayedSong(String songId);
    Single<List<PlayedSong>> getStoredPlayedSongs();
    Completable deleteStoredPlayedSong(PlayedSong playedSong);

    // Scrobbling
    Observable<ScrobbledSong> scrobblePlayedSongs(List<PlayedSong> playedSongs);
    Completable markSongAsScrobbled(PlayedSong playedSong);

    // Song Information
    Single<InfoSong> getSongInformation(ScrobbledSong scrobbledSong);
    Completable saveSongInformation(InfoSong infoSong);
}
```

**ConfigurationRepository**
**Location**: `/domain/src/main/java/com/github/niltsiar/ultimatescrobbler/domain/repository/ConfigurationRepository.java`

```java
public interface ConfigurationRepository {
    Single<UserConfiguration> getUserConfiguration();
    Completable saveUserConfiguration(UserConfiguration userConfiguration);
}
```

#### 4. Repository Implementation (Data Layer)

**ScrobblerDataRepository**
**Location**: `/data/src/main/java/com/github/niltsiar/ultimatescrobbler/data/ScrobblerDataRepository.java`

```java
public class ScrobblerDataRepository implements ScrobblerRepository {

    private final Provider<ScrobblerRemote> scrobblerRemote;  // Lazy provider
    private final ConfigurationCache configurationCache;
    private final SongsCache songsCache;

    // Mappers for converting between domain models and entities
    private final CredentialsMapper credentialsMapper;
    private final PlayedSongMapper playedSongMapper;
    private final ScrobbledSongMapper scrobbledSongMapper;
    private final InfoSongMapper infoSongMapper;

    @Inject
    public ScrobblerDataRepository(
        Provider<ScrobblerRemote> scrobblerRemote,
        ConfigurationCache configurationCache,
        SongsCache songsCache,
        CredentialsMapper credentialsMapper,
        PlayedSongMapper playedSongMapper,
        ScrobbledSongMapper scrobbledSongMapper,
        InfoSongMapper infoSongMapper) {

        this.scrobblerRemote = scrobblerRemote;
        this.configurationCache = configurationCache;
        this.songsCache = songsCache;
        this.credentialsMapper = credentialsMapper;
        this.playedSongMapper = playedSongMapper;
        this.scrobbledSongMapper = scrobbledSongMapper;
        this.infoSongMapper = infoSongMapper;
    }

    // Fetch from remote, save to cache on success
    @Override
    public Single<String> requestMobileSessionToken(Credentials credentials) {
        return scrobblerRemote.get()
            .requestMobileSessionToken(credentialsMapper.mapToEntity(credentials))
            .doOnSuccess(configurationCache.saveMobileSessionToken());
    }

    // Read from local cache
    @Override
    public Single<List<PlayedSong>> getStoredPlayedSongs() {
        return songsCache.getStoredPlayedSongs()
            .flatMapObservable(Observable::fromIterable)
            .map(playedSongMapper::mapFromEntity)  // Entity → Domain Model
            .toList();
    }

    // Write to local cache
    @Override
    public Completable savePlayedSong(PlayedSong playedSong) {
        return songsCache.savePlayedSong(playedSongMapper.mapToEntity(playedSong));
    }

    // Fetch from remote only
    @Override
    public Observable<ScrobbledSong> scrobblePlayedSongs(List<PlayedSong> playedSongs) {
        List<PlayedSongEntity> entities = new ArrayList<>();
        for (PlayedSong song : playedSongs) {
            entities.add(playedSongMapper.mapToEntity(song));
        }

        return scrobblerRemote.get()
            .scrobblePlayedSongs(entities)
            .map(scrobbledSongMapper::mapFromEntity);  // Entity → Domain Model
    }
}
```

**Coordination Strategies**:
- **Remote-first**: Authentication, scrobbling, song info fetching
- **Cache-first**: User configuration, stored played songs
- **Remote-then-cache**: Fetch from API, save to local storage on success
- **Cache-then-delete**: Scrobble from queue, delete on success

#### 5. Data Source Interfaces (Data Layer)

**ScrobblerRemote** (implemented in remote module)
**Location**: `/data/src/main/java/com/github/niltsiar/ultimatescrobbler/data/repository/ScrobblerRemote.java`

```java
public interface ScrobblerRemote {
    Single<String> requestMobileSessionToken(CredentialsEntity credentials);
    Single<ScrobbledSongEntity> updateNowPlaying(PlayedSongEntity playedSong);
    Observable<ScrobbledSongEntity> scrobblePlayedSongs(List<PlayedSongEntity> playedSongs);
    Single<InfoSongEntity> requestSongInformation(ScrobbledSongEntity scrobbledSong);
}
```

**SongsCache** (implemented in cache module)
**Location**: `/data/src/main/java/com/github/niltsiar/ultimatescrobbler/data/repository/SongsCache.java`

```java
public interface SongsCache {
    // Played Songs
    Completable savePlayedSong(PlayedSongEntity playedSong);
    Single<List<PlayedSongEntity>> getStoredPlayedSongs();
    Single<PlayedSongEntity> getStoredPlayedSong(String songId);
    Single<Long> countStoredPlayedSongs();
    Completable deleteStoredPlayedSong(PlayedSongEntity playedSong);
    Completable markSongAsScrobbled(PlayedSongEntity playedSong);

    // Current Song
    Completable saveCurrentSong(PlayedSongEntity playedSong);
    Single<PlayedSongEntity> getCurrentSong(String songId);

    // Song Info
    Completable saveSongInformation(InfoSongEntity infoSong);
}
```

**ConfigurationCache** (implemented in cache module)
**Location**: `/data/src/main/java/com/github/niltsiar/ultimatescrobbler/data/repository/ConfigurationCache.java`

```java
public interface ConfigurationCache {
    // Reads return Single<T>
    Single<String> getMobileSessionToken();
    Single<String> getUsername();
    Single<String> getPassword();
    Single<Integer> getNumberOfSongsPerBatch();
    Single<Boolean> getSendNowPlaying();

    // Writes return Consumer<T> (RxPreferences pattern)
    Consumer<? super String> saveMobileSessionToken();
    Consumer<? super String> saveUsername();
    Consumer<? super String> savePassword();
    Consumer<? super Integer> saveNumberOfSongsPerBatch();
    Consumer<? super Boolean> saveSendNowPlaying();
}
```

### Model Layers and Mapping

```
Domain Model (domain module)
      ↕ Mapper (data module)
Data Entity (data module)
      ↕ Mapper (remote/cache module)
Remote Model / Database Cursor
```

**Domain Model Example**:
**PlayedSong** (`/domain/src/main/java/com/github/niltsiar/ultimatescrobbler/domain/model/PlayedSong.java`)

```java
@AutoValue
public abstract class PlayedSong {
    public abstract String getId();
    public abstract String getTrackName();
    public abstract String getArtistName();
    public abstract String getAlbumName();
    public abstract int getLength();
    public abstract Instant getTimestamp();

    public static Builder builder() {
        return new AutoValue_PlayedSong.Builder();
    }

    @AutoValue.Builder
    public abstract static class Builder {
        public abstract Builder setId(String id);
        public abstract Builder setTrackName(String trackName);
        // ... more setters
        public abstract PlayedSong build();
    }
}
```

**Data Entity Example**:
**PlayedSongEntity** (`/data/src/main/java/com/github/niltsiar/ultimatescrobbler/data/model/PlayedSongEntity.java`)

```java
@AutoValue
public abstract class PlayedSongEntity {
    public abstract String getId();
    public abstract String getTrackName();
    public abstract String getArtistName();
    public abstract String getAlbumName();
    public abstract int getDuration();
    public abstract Instant getTimestamp();
    public abstract boolean isScrobbled();

    public static Builder builder() {
        return new AutoValue_PlayedSongEntity.Builder().setScrobbled(false);
    }

    @AutoValue.Builder
    public abstract static class Builder {
        public abstract Builder setId(String id);
        public abstract Builder setTrackName(String trackName);
        // ... more setters
        public abstract PlayedSongEntity build();
    }
}
```

**Mapper Example**:
**PlayedSongMapper** (`/data/src/main/java/com/github/niltsiar/ultimatescrobbler/data/mapper/PlayedSongMapper.java`)

```java
@Singleton
public class PlayedSongMapper implements Mapper<PlayedSongEntity, PlayedSong> {

    @Inject
    public PlayedSongMapper() {}

    @Override
    public PlayedSong mapFromEntity(PlayedSongEntity entity) {
        return PlayedSong.builder()
            .setId(entity.getId())
            .setTrackName(entity.getTrackName())
            .setArtistName(entity.getArtistName())
            .setAlbumName(entity.getAlbumName())
            .setLength(entity.getDuration())
            .setTimestamp(entity.getTimestamp())
            .build();
    }

    @Override
    public PlayedSongEntity mapToEntity(PlayedSong domainModel) {
        return PlayedSongEntity.builder()
            .setId(domainModel.getId())
            .setTrackName(domainModel.getTrackName())
            .setArtistName(domainModel.getArtistName())
            .setAlbumName(domainModel.getAlbumName())
            .setDuration(domainModel.getLength())
            .setTimestamp(domainModel.getTimestamp())
            .setScrobbled(false)
            .build();
    }
}
```

---

## Scrobbling Workflow

The scrobbling workflow is an event-driven pipeline that transforms Spotify broadcasts into Last.fm scrobbles through a series of RxJava streams and background jobs.

### Workflow Overview

```
Spotify App
    ↓ (broadcasts metadata)
SpotifyReceiver (BroadcastReceiver)
    ↓ (debounce 10s, delay 50% of song length)
ScrobblerService (Foreground Service)
    ↓ (save to database)
SQLite Database (played_songs table)
    ↓ (when batch threshold reached)
Firebase JobDispatcher
    ↓ (schedule job with network constraint)
ScrobblePlayedSongsService (JobService)
    ↓ (batch scrobble with rate limiting)
Last.fm API
    ↓ (on success)
Fetch Song Info → Save to DB → Delete from queue
```

### Step 1: Spotify Broadcast Reception

**SpotifyReceiver**
**Location**: `/app/src/main/java/com/github/niltsiar/ultimatescrobbler/receivers/SpotifyReceiver.java`

**Broadcast Intent Filter**: `com.spotify.music.metadatachanged`

**Key Components**:
```java
@Singleton
public class SpotifyReceiver extends BroadcastReceiver {

    // Event streams (RxRelay - hot observables that never complete/error)
    private final PublishRelay<PlayedSong> playedSongs = PublishRelay.create();
    private final PublishRelay<PlayedSong> nowPlaying = PublishRelay.create();
    private final PublishRelay<PlayedSong> newSong = PublishRelay.create();

    private static final int SONG_DEBOUNCE_MS = 10000;  // 10 seconds
    private static final float PERCENTAGE_TO_SCROBBLE = 0.5f;  // 50%

    @Inject
    public SpotifyReceiver() {
        // Stream 1: NEW SONG → (debounce 10s) → NOW PLAYING
        getNewSong().subscribe(nowPlaying);

        // Stream 2: NEW SONG → (debounce 10s) → (delay 50% of song) → PLAYED
        getNewSong()
            .switchMap(playedSong ->
                Observable.just(playedSong)
                    .delay(
                        (int) Math.ceil(playedSong.getLength() * PERCENTAGE_TO_SCROBBLE),
                        TimeUnit.MILLISECONDS
                    )
            )
            .subscribe(playedSongs);
    }

    @Override
    public void onReceive(Context context, Intent intent) {
        // Extract metadata from Spotify broadcast
        String trackId = intent.getStringExtra("id");
        String artistName = intent.getStringExtra("artist");
        String albumName = intent.getStringExtra("album");
        String trackName = intent.getStringExtra("track");
        int length = intent.getIntExtra("length", 0);
        long timeSent = intent.getLongExtra("timeSent", 0);

        // Discard old notifications (song already finished)
        if (System.currentTimeMillis() - timeSent > length) {
            return;
        }

        PlayedSong playedSong = PlayedSong.builder()
            .setId(trackId)
            .setArtistName(artistName)
            .setAlbumName(albumName)
            .setTrackName(trackName)
            .setLength(length)
            .setTimestamp(Instant.ofEpochMilli(timeSent))
            .build();

        // Emit to new song stream
        newSong.accept(playedSong);
    }

    // Public accessors for other components to subscribe
    public Observable<PlayedSong> getPlayedSongs() {
        return playedSongs;
    }

    public Observable<PlayedSong> getNowPlayingSong() {
        return nowPlaying;
    }

    private Observable<PlayedSong> getNewSong() {
        // Debounce prevents duplicate detections when Spotify sends rapid updates
        return newSong.debounce(SONG_DEBOUNCE_MS, TimeUnit.MILLISECONDS);
    }
}
```

**RxJava Mechanics**:

- **PublishRelay**: Hot observable that doesn't complete or error (unlike Subject)
- **Debounce**: Only emits if 10 seconds pass without a new emission (prevents duplicate song detections)
- **SwitchMap**: If a new song arrives while waiting for the delay, cancels the previous delayed emission
- **Delay Calculation**: `50% of song length` ensures user actually listened to meaningful portion

**Example Flow**:
1. User plays "Song A" (3 minutes = 180,000ms) on Spotify at timestamp T
2. Spotify broadcasts metadata intent
3. SpotifyReceiver extracts metadata
4. Emits to `newSong` relay
5. After 10-second debounce:
   - Emitted to `nowPlaying` relay (for "Now Playing" status)
   - Delayed 90 seconds (50% of 180s), then emitted to `playedSongs` relay
6. If user skips to "Song B" after 30 seconds:
   - New song emitted to `newSong`
   - Previous delay canceled (switchMap), "Song A" never reaches `playedSongs`
   - "Song B" follows same process

### Step 2: Foreground Service Coordination

**ScrobblerService**
**Location**: `/app/src/main/java/com/github/niltsiar/ultimatescrobbler/services/ScrobblerService.java`

**Lifecycle**:
- Started in `UltimateScrobblerApplication.onCreate()` as foreground service
- Runs continuously with notification
- Dynamically registers `SpotifyReceiver` for Spotify broadcasts
- Destroyed when user manually stops service

**Responsibilities**:
1. Subscribe to SpotifyReceiver event streams
2. Save played songs to local database
3. Monitor song count and trigger batch scrobbling
4. Handle "Now Playing" updates

**Implementation**:
```java
public class ScrobblerService extends Service implements HasServiceInjector {

    @Inject SpotifyReceiver spotifyReceiver;
    @Inject SavePlayedSongUseCase savePlayedSongUseCase;
    @Inject SaveCurrentSongUseCase saveCurrentSongUseCase;
    @Inject RetrieveUserConfigurationUseCase retrieveUserConfigurationUseCase;
    @Inject SendNowPlayingUseCase sendNowPlayingUseCase;
    @Inject FirebaseJobDispatcher dispatcher;

    private BehaviorRelay<Long> songCountRelay = BehaviorRelay.create();
    private CompositeDisposable disposables = new CompositeDisposable();

    @Override
    public void onCreate() {
        AndroidInjection.inject(this);
        super.onCreate();

        // Register Spotify broadcast receiver dynamically
        IntentFilter filter = new IntentFilter("com.spotify.music.metadatachanged");
        registerReceiver(spotifyReceiver, filter);

        // Start as foreground service with notification
        startForeground(NOTIFICATION_ID, createNotification());

        initialize();
    }

    private void initialize() {
        // STREAM 1: PLAYED SONGS → Save to DB → Update count
        disposables.add(
            spotifyReceiver.getPlayedSongs()
                .flatMap(playedSong ->
                    savePlayedSongUseCase.execute(playedSong)
                        .flatMapObservable(count -> Observable.just(count))
                )
                .subscribe(
                    songCountRelay::accept,  // Emit count to relay
                    Timber::e
                )
        );

        // STREAM 2: Song count changes → Check if batch threshold reached
        disposables.add(
            songCountRelay
                .flatMapSingle(count ->
                    retrieveUserConfigurationUseCase.execute(null)
                        .map(config -> new Pair<>(count, config))
                )
                .subscribe(
                    pair -> {
                        long count = pair.first;
                        UserConfiguration config = pair.second;

                        // If count >= batch size, trigger scrobble
                        if (config.getNumberOfSongsPerBatch() <= count) {
                            scrobbleNow();
                        }
                    },
                    Timber::e
                )
        );

        // STREAM 3: NOW PLAYING → Save → Schedule job (if enabled in settings)
        disposables.add(
            spotifyReceiver.getNowPlayingSong()
                .flatMapSingle(nowPlaying ->
                    retrieveUserConfigurationUseCase.execute(null)
                        .map(config -> new Pair<>(nowPlaying, config))
                )
                .subscribe(
                    pair -> {
                        PlayedSong nowPlaying = pair.first;
                        UserConfiguration config = pair.second;

                        if (config.getSendNowPlaying()) {
                            saveCurrentSongUseCase.execute(nowPlaying)
                                .subscribe(
                                    () -> {
                                        // Schedule job to send "Now Playing" to Last.fm
                                        Job job = SendNowPlayingService.createJob(
                                            dispatcher, nowPlaying.getId()
                                        );
                                        dispatcher.mustSchedule(job);
                                    },
                                    Timber::e
                                );
                        }
                    },
                    Timber::e
                )
        );
    }

    private void scrobbleNow() {
        Job scrobbleJob = ScrobblePlayedSongsService.createJob(dispatcher);
        dispatcher.mustSchedule(scrobbleJob);
    }

    @Override
    public void onDestroy() {
        unregisterReceiver(spotifyReceiver);
        disposables.clear();
        super.onDestroy();
    }
}
```

**Application Startup** (`UltimateScrobblerApplication.java`):
```java
@Override
public void onCreate() {
    super.onCreate();

    // ... Dagger setup

    // Start scrobbler service immediately
    Intent serviceIntent = new Intent(this, ScrobblerService.class);
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        startForegroundService(serviceIntent);
    } else {
        startService(serviceIntent);
    }
}
```

### Step 3: Background Job Scheduling

The app uses **Firebase JobDispatcher** for reliable background job execution with constraints (network availability, retry logic).

#### ScrobblePlayedSongsService

**Location**: `/app/src/main/java/com/github/niltsiar/ultimatescrobbler/services/ScrobblePlayedSongsService.java`

**Purpose**: Batch scrobble stored songs to Last.fm API

```java
public class ScrobblePlayedSongsService extends JobService {

    @Inject GetStoredPlayedSongsUseCase getStoredPlayedSongsUseCase;
    @Inject ScrobbleSongsUseCase scrobbleSongsUseCase;
    @Inject GetSongInformationUseCase getSongInformationUseCase;
    @Inject SaveSongInformationUseCase saveSongInformationUseCase;
    @Inject DeletePlayedSongUseCase deletePlayedSongUseCase;

    @Override
    public boolean onStartJob(JobParameters job) {
        AndroidInjection.inject(this);

        getStoredPlayedSongsUseCase.execute(null)  // Get all stored songs from DB
            .flatMapObservable(songs ->
                scrobbleSongsUseCase.execute(songs)  // Scrobble to Last.fm (Observable<ScrobbledSong>)
                    // Rate limiting: 500ms between each scrobble request
                    .zipWith(
                        Observable.interval(500, TimeUnit.MILLISECONDS),
                        (scrobbledSong, index) -> new Pair<>(songs.get(index.intValue()), scrobbledSong)
                    )
            )
            .flatMapSingle(pair ->
                getSongInformationUseCase.execute(pair.second)  // Fetch additional info from Last.fm
                    .map(infoSong -> new Pair<>(pair.first, infoSong))
            )
            .flatMap(pair ->
                saveSongInformationUseCase.execute(pair.second)  // Save song info to DB
                    .andThen(Observable.just(pair.first))
            )
            .flatMapCompletable(deletePlayedSongUseCase::execute)  // Delete from queue (scrobbled)
            .subscribe(
                () -> finishJob(job, false),  // Success - job completed
                error -> {
                    Timber.e(error);
                    finishJob(job, true);  // Failure - reschedule job
                }
            );

        return true;  // Job is still running asynchronously
    }

    @Override
    public boolean onStopJob(JobParameters job) {
        return true;  // Reschedule if job stopped prematurely
    }

    private void finishJob(JobParameters job, boolean needsReschedule) {
        jobFinished(job, needsReschedule);
    }

    public static Job createJob(FirebaseJobDispatcher dispatcher) {
        return dispatcher.newJobBuilder()
            .setService(ScrobblePlayedSongsService.class)
            .setTag("scrobble-played-songs")
            .setRecurring(false)  // One-time job
            .setLifetime(Lifetime.FOREVER)  // Persist across reboots
            .setTrigger(Trigger.NOW)  // Execute immediately (when constraints met)
            .setReplaceCurrent(false)  // Don't cancel existing job
            .setRetryStrategy(RetryStrategy.DEFAULT_LINEAR)  // Retry with linear backoff
            .setConstraints(Constraint.ON_ANY_NETWORK)  // Requires network connection
            .build();
    }
}
```

**Scrobbling Pipeline** (RxJava chain):
```
Get stored songs from DB
    ↓
Scrobble to Last.fm (Observable stream)
    ↓ (with 500ms interval for rate limiting)
Pair original song with scrobble result
    ↓
Fetch additional song info from Last.fm
    ↓
Save song info to local DB
    ↓
Delete original song from played_songs table
    ↓
Complete (or error → reschedule)
```

**Rate Limiting**:
```java
scrobbleSongsUseCase.execute(songs)
    .zipWith(
        Observable.interval(500, TimeUnit.MILLISECONDS),
        (scrobbledSong, index) -> new Pair<>(songs.get(index), scrobbledSong)
    )
```
This ensures scrobbles are sent at most once every 500ms, preventing API rate limit violations.

#### SendNowPlayingService

**Location**: `/app/src/main/java/com/github/niltsiar/ultimatescrobbler/services/SendNowPlayingService.java`

**Purpose**: Update "Now Playing" status on Last.fm (shows current song on user's profile)

```java
public class SendNowPlayingService extends JobService {

    @Inject GetCurrentSongUseCase getCurrentSongUseCase;
    @Inject SendNowPlayingUseCase sendNowPlayingUseCase;

    private static final String NOW_PLAYING_ID = "now_playing_id";

    @Override
    public boolean onStartJob(JobParameters job) {
        AndroidInjection.inject(this);

        Bundle extras = job.getExtras();
        String songId = extras.getString(NOW_PLAYING_ID);

        getCurrentSongUseCase.execute(songId)  // Fetch song from DB
            .flatMapCompletable(sendNowPlayingUseCase::execute)  // Send to Last.fm
            .subscribe(
                () -> finishJob(job, false),
                error -> {
                    Timber.e(error);
                    finishJob(job, true);
                }
            );

        return true;
    }

    @Override
    public boolean onStopJob(JobParameters job) {
        return true;
    }

    private void finishJob(JobParameters job, boolean needsReschedule) {
        jobFinished(job, needsReschedule);
    }

    public static Job createJob(FirebaseJobDispatcher dispatcher, String nowPlayingId) {
        Bundle extras = new Bundle();
        extras.putString(NOW_PLAYING_ID, nowPlayingId);

        return dispatcher.newJobBuilder()
            .setService(SendNowPlayingService.class)
            .setTag("send-now-playing")
            .setRecurring(false)
            .setLifetime(Lifetime.UNTIL_NEXT_BOOT)  // Short lifetime (not critical)
            .setTrigger(Trigger.executionWindow(0, 30))  // Execute within 30 seconds
            .setReplaceCurrent(true)  // Replace previous "now playing" job
            .setRetryStrategy(RetryStrategy.DEFAULT_LINEAR)
            .setConstraints(Constraint.ON_ANY_NETWORK)
            .setExtras(extras)
            .build();
    }
}
```

**Key Differences from Scrobble Job**:
- **Lifetime**: `UNTIL_NEXT_BOOT` (not critical if lost)
- **Replace Current**: `true` (only latest "now playing" matters)
- **Execution Window**: 30 seconds (not urgent)

### Step 4: Last.fm API Integration

#### Retrofit Service Interface

**Location**: `/remote/src/main/java/com/github/niltsiar/ultimatescrobbler/remote/ScrobblerService.java`

```java
public interface ScrobblerService {

    String WS_PATH = "";
    String RESPONSE_FORMAT = "json";

    @FormUrlEncoded
    @POST(WS_PATH)
    @Wrapped(path = {"session", "key"})  // Unwraps JSON: response.session.key
    Single<String> requestMobileSessionToken(
        @FieldMap Map<String, String> parameters,
        @Field("format") String format
    );

    @FormUrlEncoded
    @POST(WS_PATH)
    @Wrapped(path = {"nowplaying"})  // Unwraps JSON: response.nowplaying
    Single<ScrobbledSongModel> updateNowPlaying(
        @FieldMap Map<String, String> parameters,
        @Field("format") String format
    );

    @FormUrlEncoded
    @POST(WS_PATH)
    @Wrapped(path = {"scrobbles", "scrobble"})  // Unwraps: response.scrobbles.scrobble
    Observable<List<ScrobbledSongModel>> scrobbleMultiple(
        @FieldMap Map<String, String> parameters,
        @Field("format") String format
    );

    @FormUrlEncoded
    @POST(WS_PATH)
    @Wrapped(path = {"scrobbles", "scrobble"})
    Observable<ScrobbledSongModel> scrobbleSingle(
        @FieldMap Map<String, String> parameters,
        @Field("format") String format
    );

    @FormUrlEncoded
    @POST(WS_PATH)
    @Wrapped(path = {"track"})  // Unwraps: response.track
    Single<InfoSongModel> requestSongInformation(
        @FieldMap Map<String, String> parameters,
        @Field("format") String format
    );
}
```

**@Wrapped Annotation**: Moshi Lazy Adapters feature that unwraps nested JSON responses

```json
// Last.fm response:
{
  "scrobbles": {
    "scrobble": [
      { "track": "Song Name", "artist": "Artist Name", ... }
    ]
  }
}

// With @Wrapped(path = {"scrobbles", "scrobble"}), Retrofit returns:
[ { "track": "Song Name", "artist": "Artist Name", ... } ]
```

#### ScrobblerRemoteImpl

**Location**: `/remote/src/main/java/com/github/niltsiar/ultimatescrobbler/remote/ScrobblerRemoteImpl.java`

**API Signature Generation** (required by Last.fm):
```java
private String getSignature(SortedMap<String, String> params) {
    // 1. Sort parameters alphabetically
    // 2. Concatenate key+value pairs: "key1value1key2value2..."
    // 3. Append API secret
    // 4. Compute MD5 hash

    StringBuilder signatureBuilder = new StringBuilder();
    for (Map.Entry<String, String> entry : params.entrySet()) {
        signatureBuilder.append(entry.getKey()).append(entry.getValue());
    }
    signatureBuilder.append(apiSecret);

    return ByteString.encodeUtf8(signatureBuilder.toString())
        .md5()
        .hex();
}
```

**Scrobble Implementation**:
```java
@Override
public Observable<ScrobbledSongEntity> scrobblePlayedSongs(List<PlayedSongEntity> playedSongs) {
    SortedMap<String, String> params = new TreeMap<>();

    // Common parameters
    params.put("method", "track.scrobble");
    params.put("api_key", apiKey);
    params.put("sk", mobileSessionToken.get());  // Session token

    // Add indexed parameters for batch scrobbling
    // Last.fm accepts: artist[0], track[0], timestamp[0], artist[1], track[1], ...
    for (int index = 0; index < playedSongs.size(); index++) {
        PlayedSongEntity song = playedSongs.get(index);
        params.put(createIndexedParamName("artist", index), song.getArtistName());
        params.put(createIndexedParamName("track", index), song.getTrackName());
        params.put(createIndexedParamName("timestamp", index),
                   String.valueOf(song.getTimestamp().getEpochSecond()));
        params.put(createIndexedParamName("album", index), song.getAlbumName());
        params.put(createIndexedParamName("duration", index),
                   String.valueOf(song.getDuration()));
    }

    // Generate API signature
    String signature = getSignature(params);
    params.put("api_sig", signature);

    // Last.fm returns different structures for single vs. multiple scrobbles
    Observable<ScrobbledSongModel> scrobblesObservable;
    if (playedSongs.size() == 1) {
        scrobblesObservable = scrobblerService.scrobbleSingle(params, "json");
    } else {
        scrobblesObservable = scrobblerService.scrobbleMultiple(params, "json")
            .flatMap(Observable::fromIterable);
    }

    return scrobblesObservable.map(scrobbledSongMapper::mapFromRemote);
}

private String createIndexedParamName(String paramName, int index) {
    return paramName + "[" + index + "]";
}
```

**Example API Request**:
```http
POST https://ws.audioscrobbler.com/2.0
Content-Type: application/x-www-form-urlencoded

method=track.scrobble
&api_key=abc123...
&sk=session_token_here
&artist[0]=Radiohead
&track[0]=Creep
&timestamp[0]=1699123456
&album[0]=Pablo Honey
&duration[0]=238000
&artist[1]=Muse
&track[1]=Supermassive Black Hole
&timestamp[1]=1699123694
&album[1]=Black Holes and Revelations
&duration[1]=209000
&api_sig=md5_hash_here
&format=json
```

### Step 5: Local Persistence

#### Database Tables

**played_songs** (Scrobble Queue):
- Stores songs that have been played but not yet scrobbled
- Acts as offline queue
- Deleted after successful scrobble

**current_song** (Now Playing):
- Stores the currently playing song
- Updated every time a new song starts
- Used by SendNowPlayingService

**info_song** (Song Details Cache):
- Stores additional song information fetched from Last.fm
- Includes: album art URL, tags, wiki content
- Used by song details screen

#### SongsCacheImpl

**Location**: `/cache/src/main/java/com/github/niltsiar/ultimatescrobbler/cache/storage/SongsCacheImpl.java`

```java
@Singleton
public class SongsCacheImpl implements SongsCache {

    private final Context context;
    private final PlayedSongEntityMapper playedSongMapper;
    private final InfoSongEntityMapper infoSongMapper;

    @Inject
    public SongsCacheImpl(Context context,
                         PlayedSongEntityMapper playedSongMapper,
                         InfoSongEntityMapper infoSongMapper) {
        this.context = context;
        this.playedSongMapper = playedSongMapper;
        this.infoSongMapper = infoSongMapper;
    }

    @Override
    public Completable savePlayedSong(PlayedSongEntity playedSong) {
        return Completable.fromAction(() -> {
            ContentValues values = playedSongMapper.mapToContentValues(playedSong);
            context.getContentResolver().insert(
                SongsProvider.PlayedSongs.CONTENT_URI,
                values
            );
        });
    }

    @Override
    public Single<List<PlayedSongEntity>> getStoredPlayedSongs() {
        return Single.fromCallable(() -> {
            Cursor cursor = context.getContentResolver().query(
                SongsProvider.PlayedSongs.CONTENT_URI,
                null,  // All columns
                PlayedSongColumns.SCROBBLED + " = ?",  // WHERE scrobbled = 0
                new String[]{"0"},
                PlayedSongColumns.TIMESTAMP + " ASC"  // ORDER BY timestamp
            );

            List<PlayedSongEntity> songs = new ArrayList<>();
            if (cursor != null) {
                while (cursor.moveToNext()) {
                    songs.add(playedSongMapper.mapFromCursor(cursor));
                }
                cursor.close();
            }
            return songs;
        });
    }

    @Override
    public Completable deleteStoredPlayedSong(PlayedSongEntity playedSong) {
        return Completable.fromAction(() -> {
            context.getContentResolver().delete(
                SongsProvider.PlayedSongs.CONTENT_URI,
                PlayedSongColumns.ID + " = ?",
                new String[]{playedSong.getId()}
            );
        });
    }
}
```

### Complete Flow Example

**User plays "Bohemian Rhapsody" by Queen (354 seconds) on Spotify**:

1. **T+0s**: Spotify broadcasts metadata intent
2. **T+0s**: `SpotifyReceiver.onReceive()` extracts metadata
3. **T+10s**: Debounce period ends
   - Emitted to `nowPlaying` stream
   - Delayed by 177s (50% of 354s)
4. **T+10s**: `ScrobblerService` receives "now playing" event
   - Saves to `current_song` table
   - Schedules `SendNowPlayingService` job
5. **T+10-40s**: `SendNowPlayingService` executes
   - Sends "Now Playing" to Last.fm API
   - Last.fm shows "Queen - Bohemian Rhapsody" on user's profile
6. **T+187s**: Delay period ends (user listened to 50%)
   - Emitted to `playedSongs` stream
7. **T+187s**: `ScrobblerService` receives "played" event
   - Saves to `played_songs` table (queue)
   - Returns new count (e.g., 1)
8. **T+187s**: If count < batch threshold (e.g., 10):
   - Song stays in queue
   - User plays 9 more songs...
9. **When 10th song added**: Batch threshold reached
   - `ScrobblePlayedSongsService` job scheduled
10. **T+???**: Job executes (when network available)
    - Fetches 10 songs from `played_songs` table
    - Scrobbles batch to Last.fm (500ms intervals)
    - Fetches additional info for each song
    - Saves info to `info_song` table
    - Deletes songs from `played_songs` table
    - Job complete

**Resilience Features**:
- **Offline Support**: Songs queued locally until network available
- **Retry Logic**: Failed jobs rescheduled with linear backoff
- **Crash Recovery**: `Lifetime.FOREVER` ensures jobs survive app restarts
- **Duplicate Prevention**: Debouncing + switchMap prevent duplicate scrobbles
- **Rate Limiting**: 500ms delay between API requests

---

## Key Patterns and Components

### 1. MVVM (Model-View-ViewModel)

The app uses MVVM with Android Architecture Components for a clear separation between UI and business logic.

#### ViewModel

ViewModels survive configuration changes (e.g., screen rotation) and manage UI-related data.

**Example**: `PlayedSongsViewModel`
**Location**: `/app/src/main/java/com/github/niltsiar/ultimatescrobbler/ui/songs/playedsongs/PlayedSongsViewModel.java`

```java
public class PlayedSongsViewModel extends ViewModel {

    private final GetPlayedSongsUseCase getPlayedSongsUseCase;
    private final DeletePlayedSongUseCase deletePlayedSongUseCase;

    // RxRelay for ViewState stream (hot observable)
    private final BehaviorRelay<PlayedSongsViewState> playedSongsViewStateRelay =
        BehaviorRelay.create();

    private final CompositeDisposable disposables = new CompositeDisposable();

    @Inject
    public PlayedSongsViewModel(
        GetPlayedSongsUseCase getPlayedSongsUseCase,
        DeletePlayedSongUseCase deletePlayedSongUseCase) {

        this.getPlayedSongsUseCase = getPlayedSongsUseCase;
        this.deletePlayedSongUseCase = deletePlayedSongUseCase;
    }

    public void loadPlayedSongs() {
        disposables.add(
            getPlayedSongsUseCase.execute(null)
                .map(songs -> {
                    if (songs.isEmpty()) {
                        return PlayedSongsViewState.empty();
                    } else {
                        return PlayedSongsViewState.withSongs(songs);
                    }
                })
                .subscribe(
                    playedSongsViewStateRelay::accept,
                    error -> playedSongsViewStateRelay.accept(
                        PlayedSongsViewState.error(error.getMessage())
                    )
                )
        );
    }

    public void deleteSong(PlayedSong song) {
        disposables.add(
            deletePlayedSongUseCase.execute(song)
                .subscribe(
                    this::loadPlayedSongs,  // Reload after delete
                    Timber::e
                )
        );
    }

    public Observable<PlayedSongsViewState> getViewState() {
        return playedSongsViewStateRelay;
    }

    @Override
    protected void onCleared() {
        disposables.clear();  // Prevent memory leaks
        super.onCleared();
    }
}
```

#### ViewState

Immutable data classes representing complete UI state.

**PlayedSongsViewState**:
```java
@AutoValue
public abstract class PlayedSongsViewState {

    public enum State {
        LOADING,
        LOADED,
        EMPTY,
        ERROR
    }

    public abstract State getState();
    @Nullable public abstract List<PlayedSong> getSongs();
    @Nullable public abstract String getErrorMessage();

    public static PlayedSongsViewState loading() {
        return new AutoValue_PlayedSongsViewState(State.LOADING, null, null);
    }

    public static PlayedSongsViewState withSongs(List<PlayedSong> songs) {
        return new AutoValue_PlayedSongsViewState(State.LOADED, songs, null);
    }

    public static PlayedSongsViewState empty() {
        return new AutoValue_PlayedSongsViewState(State.EMPTY, null, null);
    }

    public static PlayedSongsViewState error(String message) {
        return new AutoValue_PlayedSongsViewState(State.ERROR, null, message);
    }
}
```

#### ViewModelFactory

Custom factory for injecting dependencies into ViewModels.

```java
public class PlayedSongsViewModelFactory implements ViewModelProvider.Factory {

    private final GetPlayedSongsUseCase getPlayedSongsUseCase;
    private final DeletePlayedSongUseCase deletePlayedSongUseCase;

    @Inject
    public PlayedSongsViewModelFactory(
        GetPlayedSongsUseCase getPlayedSongsUseCase,
        DeletePlayedSongUseCase deletePlayedSongUseCase) {

        this.getPlayedSongsUseCase = getPlayedSongsUseCase;
        this.deletePlayedSongUseCase = deletePlayedSongUseCase;
    }

    @Override
    public <T extends ViewModel> T create(Class<T> modelClass) {
        if (modelClass.isAssignableFrom(PlayedSongsViewModel.class)) {
            return (T) new PlayedSongsViewModel(
                getPlayedSongsUseCase,
                deletePlayedSongUseCase
            );
        }
        throw new IllegalArgumentException("Unknown ViewModel class");
    }
}
```

#### Fragment (View)

```java
public class PlayedSongsFragment extends Fragment {

    @Inject PlayedSongsViewModelFactory viewModelFactory;
    private PlayedSongsViewModel viewModel;

    @BindView(R.id.recycler_view) RecyclerView recyclerView;
    private PlayedSongsAdapter adapter;

    private CompositeDisposable disposables = new CompositeDisposable();

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        AndroidSupportInjection.inject(this);  // Dagger injection

        viewModel = ViewModelProviders.of(this, viewModelFactory)
            .get(PlayedSongsViewModel.class);
    }

    @Override
    public void onResume() {
        super.onResume();

        // Subscribe to ViewState
        disposables.add(
            viewModel.getViewState()
                .subscribe(this::render, Timber::e)
        );

        viewModel.loadPlayedSongs();
    }

    private void render(PlayedSongsViewState viewState) {
        switch (viewState.getState()) {
            case LOADING:
                showLoading();
                break;
            case LOADED:
                showSongs(viewState.getSongs());
                break;
            case EMPTY:
                showEmpty();
                break;
            case ERROR:
                showError(viewState.getErrorMessage());
                break;
        }
    }

    @Override
    public void onPause() {
        disposables.clear();
        super.onPause();
    }
}
```

### 2. RxJava Patterns

#### Use Case Base Classes

All Use Cases extend one of three base classes that handle threading:

```java
// For streams (multiple emissions)
public abstract class ObservableUseCase<T, V> {
    protected abstract Observable<T> buildUseCaseObservable(@Nullable V param);

    public Observable<T> execute(@Nullable V param) {
        return buildUseCaseObservable(param)
            .subscribeOn(executionScheduler)       // I/O thread
            .observeOn(postExecutionScheduler);    // Main thread
    }
}

// For single result
public abstract class SingleUseCase<T, V> {
    protected abstract Single<T> buildUseCaseObservable(@Nullable V param);

    public Single<T> execute(@Nullable V param) {
        return buildUseCaseObservable(param)
            .subscribeOn(executionScheduler)
            .observeOn(postExecutionScheduler);
    }
}

// For side effects (no result)
public abstract class CompletableUseCase<T> {
    protected abstract Completable buildUseCaseObservable(@Nullable T param);

    public Completable execute(@Nullable T param) {
        return buildUseCaseObservable(param)
            .subscribeOn(executionScheduler)
            .observeOn(postExecutionScheduler);
    }
}
```

**Scheduler Injection** (ApplicationModule):
```java
@Provides
@Singleton
GetPlayedSongsUseCase provideGetPlayedSongsUseCase(ScrobblerRepository repository) {
    return new GetPlayedSongsUseCase(
        repository,
        Schedulers.io(),               // Background execution
        AndroidSchedulers.mainThread()  // UI observation
    );
}
```

#### RxRelay for Event Streams

**RxRelay** is used instead of Subject/Observable for event streams because it:
- Never completes or errors (hot observable)
- Prevents accidental termination of streams
- Thread-safe

**PublishRelay** (no initial value, only new emissions):
```java
private final PublishRelay<PlayedSong> playedSongs = PublishRelay.create();

public Observable<PlayedSong> getPlayedSongs() {
    return playedSongs;  // Consumers subscribe
}

// Producer emits
playedSongs.accept(newSong);
```

**BehaviorRelay** (replays last value to new subscribers):
```java
private final BehaviorRelay<ConfigurationViewState> viewStateRelay =
    BehaviorRelay.create();

public Observable<ConfigurationViewState> getViewState() {
    return viewStateRelay;
}

// New subscribers immediately receive last emitted ViewState
viewStateRelay.accept(newViewState);
```

#### Disposable Management

All RxJava subscriptions must be disposed to prevent memory leaks.

```java
// ViewModel
private final CompositeDisposable disposables = new CompositeDisposable();

public void loadData() {
    disposables.add(
        useCase.execute(param)
            .subscribe(this::handleSuccess, Timber::e)
    );
}

@Override
protected void onCleared() {
    disposables.clear();  // Dispose all subscriptions
    super.onCleared();
}
```

```java
// Activity/Fragment
private final CompositeDisposable disposables = new CompositeDisposable();

@Override
protected void onResume() {
    super.onResume();
    disposables.add(
        viewModel.getViewState().subscribe(this::render)
    );
}

@Override
protected void onPause() {
    disposables.clear();
    super.onPause();
}
```

### 3. Repository Pattern

Repositories abstract data sources and coordinate between remote and local storage.

**Interface in Domain Layer**:
```java
public interface ScrobblerRepository {
    Single<List<PlayedSong>> getStoredPlayedSongs();
    Completable savePlayedSong(PlayedSong playedSong);
    Observable<ScrobbledSong> scrobblePlayedSongs(List<PlayedSong> playedSongs);
}
```

**Implementation in Data Layer**:
```java
public class ScrobblerDataRepository implements ScrobblerRepository {

    private final Provider<ScrobblerRemote> scrobblerRemote;
    private final SongsCache songsCache;
    private final PlayedSongMapper mapper;

    @Override
    public Single<List<PlayedSong>> getStoredPlayedSongs() {
        // Cache-only
        return songsCache.getStoredPlayedSongs()
            .flatMapObservable(Observable::fromIterable)
            .map(mapper::mapFromEntity)
            .toList();
    }

    @Override
    public Completable savePlayedSong(PlayedSong playedSong) {
        // Cache-only
        return songsCache.savePlayedSong(mapper.mapToEntity(playedSong));
    }

    @Override
    public Observable<ScrobbledSong> scrobblePlayedSongs(List<PlayedSong> playedSongs) {
        // Remote-only
        List<PlayedSongEntity> entities = new ArrayList<>();
        for (PlayedSong song : playedSongs) {
            entities.add(mapper.mapToEntity(song));
        }
        return scrobblerRemote.get().scrobblePlayedSongs(entities)
            .map(scrobbledSongMapper::mapFromEntity);
    }
}
```

### 4. AutoValue for Immutability

All models use **AutoValue** for immutable value objects with builder pattern.

```java
@AutoValue
public abstract class PlayedSong {

    public abstract String getId();
    public abstract String getTrackName();
    public abstract String getArtistName();
    public abstract String getAlbumName();
    public abstract int getLength();
    public abstract Instant getTimestamp();

    public static Builder builder() {
        return new AutoValue_PlayedSong.Builder();  // Generated class
    }

    @AutoValue.Builder
    public abstract static class Builder {
        public abstract Builder setId(String id);
        public abstract Builder setTrackName(String trackName);
        public abstract Builder setArtistName(String artistName);
        public abstract Builder setAlbumName(String albumName);
        public abstract Builder setLength(int length);
        public abstract Builder setTimestamp(Instant timestamp);
        public abstract PlayedSong build();
    }
}
```

**Usage**:
```java
PlayedSong song = PlayedSong.builder()
    .setId("123")
    .setTrackName("Creep")
    .setArtistName("Radiohead")
    .setAlbumName("Pablo Honey")
    .setLength(238000)
    .setTimestamp(Instant.now())
    .build();

// Immutable - must create new instance for changes
PlayedSong renamed = song.toBuilder()
    .setTrackName("New Name")
    .build();
```

### 5. Database with Schematic

**Schematic** generates ContentProvider boilerplate from annotations.

**Database Definition**:
```java
@Database(version = 1, packageName = "com.github.niltsiar.ultimatescrobbler.cache.provider")
public class SongsDatabase {

    @Table(PlayedSongColumns.class)
    public static final String PLAYED_SONGS = "played_songs";

    @Table(InfoSongColumns.class)
    public static final String INFO_SONG = "info_song";

    @Table(PlayedSongColumns.class)
    public static final String CURRENT_SONG = "current_song";
}
```

**Column Definition**:
```java
public interface PlayedSongColumns {
    @DataType(DataType.Type.TEXT)
    @PrimaryKey(onConflict = ConflictResolutionType.REPLACE)
    String ID = "_id";

    @DataType(DataType.Type.TEXT)
    @NotNull
    String TRACK_NAME = "track_name";

    @DataType(DataType.Type.TEXT)
    @NotNull
    String ARTIST_NAME = "artist_name";

    @DataType(DataType.Type.INTEGER)
    @NotNull
    String TIMESTAMP = "played_instant";
}
```

**Generated ContentProvider** (`SongsProvider`):
```java
// Auto-generated at compile time
public class SongsProvider extends ContentProvider {

    public static class PlayedSongs {
        public static final Uri CONTENT_URI =
            Uri.parse("content://com.github.niltsiar.ultimatescrobbler.cache.provider/played_songs");
    }

    // CRUD operations auto-generated
}
```

**Usage**:
```java
// Insert
ContentValues values = new ContentValues();
values.put(PlayedSongColumns.ID, "123");
values.put(PlayedSongColumns.TRACK_NAME, "Creep");
context.getContentResolver().insert(SongsProvider.PlayedSongs.CONTENT_URI, values);

// Query
Cursor cursor = context.getContentResolver().query(
    SongsProvider.PlayedSongs.CONTENT_URI,
    null,  // All columns
    null,  // No WHERE clause
    null,
    PlayedSongColumns.TIMESTAMP + " DESC"  // ORDER BY
);
```

### 6. Services and BroadcastReceivers

#### ScrobblerService (Foreground Service)

**Purpose**: Long-running background service that coordinates scrobbling workflow

**Features**:
- Runs as foreground service with persistent notification
- Dynamically registers `SpotifyReceiver`
- Subscribes to song event streams
- Manages scrobble queue
- Schedules background jobs

**Lifecycle**:
```
Application.onCreate()
    ↓
startForegroundService(ScrobblerService)
    ↓
ScrobblerService.onCreate()
    ↓
Register SpotifyReceiver
    ↓
Start foreground (notification)
    ↓
Subscribe to event streams
    ↓
Service runs until stopped
```

#### SpotifyReceiver (BroadcastReceiver)

**Purpose**: Detect songs played on Spotify via broadcasts

**Broadcast Intent**: `com.spotify.music.metadatachanged`

**Extracted Metadata**:
- `id` - Spotify track ID
- `artist` - Artist name
- `album` - Album name
- `track` - Track name
- `length` - Duration in milliseconds
- `timeSent` - Timestamp when broadcast was sent

**Event Streams**:
- `newSong` - Raw song detections (debounced 10s)
- `nowPlaying` - Songs to update "Now Playing" status
- `playedSongs` - Songs that have been listened to 50%+

#### ScrobblePlayedSongsService (JobService)

**Purpose**: Background job for batch scrobbling to Last.fm

**Constraints**:
- Requires network connection (`Constraint.ON_ANY_NETWORK`)
- Retry strategy: Linear backoff
- Lifetime: `FOREVER` (survives reboots)

**Workflow**:
1. Fetch stored songs from database
2. Scrobble to Last.fm (rate limited: 500ms intervals)
3. Fetch additional song info
4. Save song info to database
5. Delete scrobbled songs from queue

#### SendNowPlayingService (JobService)

**Purpose**: Update "Now Playing" status on Last.fm

**Constraints**:
- Requires network connection
- Execution window: 0-30 seconds
- Lifetime: `UNTIL_NEXT_BOOT` (not critical)
- Replace current: `true` (only latest matters)

---

## Package Structure

### App Module (`/app/src/main/java/com/github/niltsiar/ultimatescrobbler/`)

```
app/
├── UltimateScrobblerApplication.java       # Application class (Dagger setup)
│
├── di/                                     # Dependency Injection
│   ├── ApplicationComponent.java          # Root Dagger component
│   └── module/
│       ├── ApplicationModule.java         # Core dependencies, Use Cases
│       ├── ActivityBindingModule.java     # Activity injection
│       ├── FragmentBindingModule.java     # Fragment injection
│       ├── ServiceBindingModule.java      # Service injection
│       └── NetworkModule.java             # OkHttp (debug/release variants)
│
├── receivers/                             # Broadcast Receivers
│   └── SpotifyReceiver.java              # Spotify metadata detection
│
├── services/                              # Services
│   ├── ScrobblerService.java             # Foreground service (coordination)
│   ├── ScrobblePlayedSongsService.java   # Job service (batch scrobble)
│   └── SendNowPlayingService.java        # Job service (now playing)
│
├── ui/                                    # User Interface
│   ├── configuration/                     # Configuration screen
│   │   ├── ConfigurationActivity.java
│   │   ├── ConfigurationViewModel.java
│   │   ├── ConfigurationViewModelFactory.java
│   │   ├── ConfigurationViewState.java
│   │   └── ConfigurationViewStateMapper.java
│   │
│   ├── songdetails/                       # Song details screen
│   │   ├── SongDetailsActivity.java
│   │   ├── SongDetailsViewModel.java
│   │   └── SongDetailsViewModelFactory.java
│   │
│   └── songs/                             # Songs list screen
│       ├── SongsActivity.java             # Main activity (tabs)
│       │
│       ├── playedsongs/                   # Played songs tab
│       │   ├── PlayedSongsFragment.java
│       │   ├── PlayedSongsViewModel.java
│       │   ├── PlayedSongsViewModelFactory.java
│       │   ├── PlayedSongsAdapter.java    # RecyclerView adapter
│       │   └── PlayedSongItemViewHolder.java
│       │
│       └── scrobbledsongs/                # Scrobbled songs tab
│           ├── ScrobbledSongsFragment.java
│           ├── ScrobbledSongsViewModel.java
│           ├── ScrobbledSongsViewModelFactory.java
│           ├── ScrobbledSongsAdapter.java
│           ├── ScrobbledSongItemViewHolder.java
│           └── ScrobbledSongItemClickedState.java
│
├── utils/                                 # Utilities
│   ├── LoaderProvider.java
│   ├── Utils.java
│   └── rxindicatorseekbar/                # RxJava bindings for SeekBar
│       ├── RxIndicatorSeekBar.java
│       └── IndicatorSeekBarChangeObservable.java
│
└── widget/                                # App Widget
    ├── UltimateScrobblerWidget.java
    └── UltimateScrobblerWidgetService.java
```

### Domain Module (`/domain/src/main/java/com/github/niltsiar/ultimatescrobbler/domain/`)

```
domain/
├── error/                                 # Domain errors
│   └── InvalidCredentialsError.java
│
├── interactor/                            # Use Cases
│   ├── ObservableUseCase.java            # Base class for streams
│   ├── SingleUseCase.java                # Base class for single result
│   ├── CompletableUseCase.java           # Base class for side effects
│   │
│   ├── configuration/                     # User configuration use cases
│   │   ├── RetrieveUserConfigurationUseCase.java
│   │   └── SaveUserConfigurationUseCase.java
│   │
│   ├── mobilesession/                     # Authentication use cases
│   │   └── RequestMobileSessionTokenUseCase.java
│   │
│   ├── playedsong/                        # Played song use cases
│   │   ├── GetPlayedSongsUseCase.java
│   │   ├── GetPlayedSongUseCase.java
│   │   ├── GetCurrentSongUseCase.java
│   │   ├── SavePlayedSongUseCase.java
│   │   ├── SaveCurrentSongUseCase.java
│   │   ├── DeletePlayedSongUseCase.java
│   │   ├── ScrobbleSongsUseCase.java
│   │   └── SendNowPlayingUseCase.java
│   │
│   └── songinformation/                   # Song info use cases
│       ├── GetSongInformationUseCase.java
│       └── SaveSongInformationUseCase.java
│
├── model/                                 # Domain Models (AutoValue)
│   ├── PlayedSong.java                   # Song that was played
│   ├── ScrobbledSong.java                # Song scrobbled to Last.fm
│   ├── InfoSong.java                     # Detailed song information
│   ├── Credentials.java                  # User credentials
│   └── UserConfiguration.java            # User settings
│
├── repository/                            # Repository Interfaces
│   ├── ScrobblerRepository.java          # Scrobbling operations
│   └── ConfigurationRepository.java      # Configuration operations
│
└── utils/
    └── RxBus.java                        # Event bus (if used)
```

### Data Module (`/data/src/main/java/com/github/niltsiar/ultimatescrobbler/data/`)

```
data/
├── ScrobblerDataRepository.java          # ScrobblerRepository implementation
├── UserConfigurationDataRepository.java  # ConfigurationRepository implementation
│
├── mapper/                               # Entity ↔ Domain Model mappers
│   ├── Mapper.java                       # Mapper interface
│   ├── PlayedSongMapper.java
│   ├── ScrobbledSongMapper.java
│   ├── InfoSongMapper.java
│   ├── CredentialsMapper.java
│   └── UserConfigurationMapper.java
│
├── model/                                # Data Entities (AutoValue)
│   ├── PlayedSongEntity.java
│   ├── ScrobbledSongEntity.java
│   ├── InfoSongEntity.java
│   ├── CredentialsEntity.java
│   └── UserConfigurationEntity.java
│
└── repository/                           # Data Source Interfaces
    ├── ScrobblerRemote.java              # Remote data source contract
    ├── SongsCache.java                   # Local songs storage contract
    └── ConfigurationCache.java           # Preferences storage contract
```

### Remote Module (`/remote/src/main/java/com/github/niltsiar/ultimatescrobbler/remote/`)

```
remote/
├── ScrobblerService.java                 # Retrofit service interface
├── ScrobblerRemoteImpl.java             # ScrobblerRemote implementation
│
├── mapper/                               # Remote Model ↔ Entity mappers
│   ├── EntityMapper.java                 # Mapper interface
│   ├── ScrobbledSongMapper.java
│   └── InfoSongMapper.java
│
├── model/                                # Remote Models (AutoValue + Moshi)
│   ├── ScrobbledSongModel.java
│   ├── InfoSongModel.java
│   └── AutoValueMoshiAdapterFactory.java # Moshi adapter factory
│
└── qualifiers/                           # Dagger qualifiers
    ├── ApiKey.java                       # @ApiKey qualifier
    ├── ApiSecret.java                    # @ApiSecret qualifier
    └── MobileSessionToken.java           # @MobileSessionToken qualifier
```

### Cache Module (`/cache/src/main/java/com/github/niltsiar/ultimatescrobbler/cache/`)

```
cache/
├── database/                             # SQLite database (Schematic)
│   ├── SongsDatabase.java                # @Database definition
│   ├── PlayedSongColumns.java            # played_songs table schema
│   ├── InfoSongColumns.java              # info_song table schema
│   └── SongsProvider.java                # Generated ContentProvider
│
├── preferences/                          # SharedPreferences
│   └── ConfigurationCacheImpl.java       # ConfigurationCache implementation
│
├── storage/                              # Database operations
│   └── SongsCacheImpl.java               # SongsCache implementation
│
└── mapper/                               # Cursor/ContentValues ↔ Entity mappers
    ├── PlayedSongEntityMapper.java
    └── InfoSongEntityMapper.java
```

---

## Summary

Ultimate Scrobbler demonstrates a well-structured Clean Architecture implementation on Android with the following key characteristics:

### Architectural Strengths

1. **Clear Separation of Concerns**: Five independent modules with well-defined responsibilities
2. **Dependency Rule Adherence**: Dependencies point inward, with domain layer having zero Android dependencies
3. **Testability**: Pure Java domain layer can be tested without Android framework
4. **Reactive Programming**: RxJava2 throughout for asynchronous operations and event streams
5. **MVVM Pattern**: ViewModels manage UI state with reactive ViewState streams
6. **Dependency Injection**: Dagger 2 for compile-time dependency graph validation
7. **Immutability**: AutoValue for immutable value objects
8. **Event-Driven**: Spotify broadcasts → RxRelay streams → Background jobs → API

### Data Flow

```
UI (Activities/Fragments/ViewModels)
    ↓ observes ViewState via RxRelay
    ↓ calls Use Cases
Domain (Use Cases with RxJava schedulers)
    ↓ calls Repository interfaces
    ↓ operates on Domain Models
Data (Repository implementations)
    ↓ coordinates Remote + Cache
    ↓ maps Entities ↔ Domain Models
Remote (Last.fm API) + Cache (SQLite + SharedPreferences)
```

### Scrobbling Pipeline

```
Spotify → SpotifyReceiver → (debounce + delay) → ScrobblerService
    ↓ save to DB
SQLite Queue → (batch threshold) → Firebase JobDispatcher
    ↓ network available
ScrobblePlayedSongsService → (rate limited) → Last.fm API
    ↓ success
Fetch Info → Save to DB → Delete from Queue
```

### Key Technologies

- **Clean Architecture**: Multi-module separation (app, domain, data, remote, cache)
- **Dagger 2**: Dependency injection with AndroidInjector
- **RxJava2**: Reactive streams and threading
- **MVVM**: ViewModels + ViewStates + Architecture Components
- **Retrofit**: Last.fm API integration
- **Schematic**: ContentProvider code generation
- **Firebase JobDispatcher**: Reliable background jobs
- **AutoValue**: Immutable value objects
- **RxPreferences**: Reactive SharedPreferences

### Design Patterns

- **Repository Pattern**: Abstract data sources behind interfaces
- **Use Case Pattern**: Single-responsibility business operations
- **Factory Pattern**: ViewModelFactory for ViewModel creation
- **Observer Pattern**: RxJava observables and RxRelay
- **Builder Pattern**: AutoValue builders for immutable objects
- **Dependency Injection**: Dagger 2 for inversion of control
- **MVVM**: Separation of UI and business logic

This architecture provides a solid foundation for scalability, maintainability, and testability while handling the complex event-driven nature of music scrobbling.
