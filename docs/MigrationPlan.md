# Migration Plan: Feature-Based Clean Architecture with Typed Errors

## Executive Summary

This document outlines a comprehensive migration plan for the Ultimate Scrobbler Android application, transitioning from the current horizontal layer-based architecture to a modern feature-based vertical slicing architecture with typed error handling.

### Current State
- **Architecture**: Clean Architecture with horizontal slicing (5 modules: app, domain, data, remote, cache)
- **Language**: Java 8
- **Async**: RxJava2 (Single, Observable, Completable)
- **Error Handling**: Untyped exceptions with Timber logging
- **DI**: Dagger 2.14.1
- **Database**: Schematic (ContentProvider generation)

### Target State
- **Architecture**: Clean Architecture with vertical feature slicing + API/Implementation separation
- **Language**: Kotlin 1.9+
- **Async**: Kotlin Coroutines + Flow
- **Error Handling**: Arrow Either with typed errors + raise DSL
- **DI**: Hilt (Dagger for Android)
- **Database**: Room (modern SQLite wrapper)

### Migration Goals

1. ✅ **Maintain Clean Architecture** principles (dependency rule, separation of concerns)
2. ✅ **Feature-based modules** for better scalability and team autonomy
3. ✅ **Separate API and Implementation modules** for each feature
4. ✅ **Typed error handling** with `Either<Error, Success>` pattern
5. ✅ **Simplified Either composition** using Arrow's `raise` DSL

---

## Table of Contents

1. [New Architecture Overview](#new-architecture-overview)
2. [Module Structure](#module-structure)
3. [Typed Error Hierarchy](#typed-error-hierarchy)
4. [Either and Raise Pattern](#either-and-raise-pattern)
5. [Migration Phases](#migration-phases)
6. [Technology Stack Changes](#technology-stack-changes)
7. [Code Examples](#code-examples)
8. [Risks and Mitigation](#risks-and-mitigation)
9. [Timeline and Resources](#timeline-and-resources)
10. [Success Criteria](#success-criteria)

---

## New Architecture Overview

### Architectural Principles

The new architecture maintains Clean Architecture principles while shifting from **horizontal slicing** (by layer) to **vertical slicing** (by feature):

```
┌─────────────────────────────────────────────────────────────┐
│                         :app                                │
│  (UI Layer, Composition Root, DI Setup)                     │
└──────────┬──────────────────────────────────────────────────┘
           │ depends on all feature-api modules
           ↓
┌──────────────────────────────────────────────────────────────┐
│              Feature Modules (Vertical Slices)              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────┐  │
│  │ feature-auth    │  │ feature-scrobble│  │ feature-   │  │
│  ├─────────────────┤  ├─────────────────┤  │ songs      │  │
│  │ auth-api        │  │ scrobble-api    │  ├────────────┤  │
│  │ auth-impl       │  │ scrobble-impl   │  │ songs-api  │  │
│  └─────────────────┘  └─────────────────┘  │ songs-impl │  │
│                                             └────────────┘  │
│                                                              │
│  Each feature is self-contained with:                       │
│  - Domain models                                            │
│  - Use cases                                                │
│  - Repository interfaces                                    │
│  - Repository implementations                               │
│  - Data sources (Remote/Cache)                              │
└──────────────────────────────────────────────────────────────┘
           │ depends on
           ↓
┌──────────────────────────────────────────────────────────────┐
│                    :core modules                            │
├──────────────────────────────────────────────────────────────┤
│  :core-api     - Shared interfaces, base error types        │
│  :core-impl    - Network, database, utilities               │
└──────────────────────────────────────────────────────────────┘
```

### Benefits of Feature Modules

1. **Team Scalability**: Different teams can work on different features independently
2. **Compile Time**: Smaller modules compile faster
3. **Clear Boundaries**: Feature boundaries are explicit
4. **Easier Testing**: Features can be tested in isolation
5. **Code Ownership**: Clear ownership per feature
6. **Discoverability**: Easy to find all code related to a feature

### Benefits of API/Implementation Separation

1. **Abstraction**: Implementation details hidden behind interfaces
2. **Testing**: Easy to mock implementations in tests
3. **Flexibility**: Can swap implementations without changing consumers
4. **Dependency Management**: API modules have minimal dependencies
5. **Contract-First**: API defines the contract, multiple implementations possible

---

## Module Structure

### Complete Module Layout

```
android_nanodegree_capstone/
│
├── app/
│   ├── src/main/java/.../
│   │   ├── UltimateScrobblerApp.kt
│   │   ├── di/
│   │   │   └── AppModule.kt
│   │   └── ui/
│   │       ├── configuration/
│   │       ├── songs/
│   │       └── songdetails/
│   └── build.gradle.kts
│
├── core/
│   ├── core-api/
│   │   ├── src/main/java/.../core/
│   │   │   ├── error/
│   │   │   │   └── DomainError.kt
│   │   │   ├── model/
│   │   │   │   └── (shared domain models if any)
│   │   │   └── util/
│   │   │       ├── Either.kt (re-export from Arrow)
│   │   │       └── Result.kt
│   │   └── build.gradle.kts
│   │
│   └── core-impl/
│       ├── src/main/java/.../core/
│       │   ├── database/
│       │   │   ├── AppDatabase.kt
│       │   │   └── DatabaseModule.kt
│       │   ├── network/
│       │   │   ├── NetworkModule.kt
│       │   │   └── interceptors/
│       │   └── preferences/
│       │       └── PreferencesManager.kt
│       └── build.gradle.kts
│
├── features/
│   │
│   ├── feature-auth/
│   │   ├── auth-api/
│   │   │   ├── src/main/java/.../auth/api/
│   │   │   │   ├── model/
│   │   │   │   │   ├── Credentials.kt
│   │   │   │   │   ├── Session.kt
│   │   │   │   │   └── UserProfile.kt
│   │   │   │   ├── repository/
│   │   │   │   │   └── AuthRepository.kt
│   │   │   │   ├── usecase/
│   │   │   │   │   ├── LoginUseCase.kt
│   │   │   │   │   ├── LogoutUseCase.kt
│   │   │   │   │   └── GetSessionUseCase.kt
│   │   │   │   └── error/
│   │   │   │       └── AuthError.kt
│   │   │   └── build.gradle.kts
│   │   │
│   │   └── auth-impl/
│   │       ├── src/main/java/.../auth/impl/
│   │       │   ├── di/
│   │       │   │   └── AuthModule.kt
│   │       │   ├── repository/
│   │       │   │   └── AuthRepositoryImpl.kt
│   │       │   ├── usecase/
│   │       │   │   ├── LoginUseCaseImpl.kt
│   │       │   │   └── LogoutUseCaseImpl.kt
│   │       │   ├── remote/
│   │       │   │   ├── AuthApi.kt
│   │       │   │   ├── AuthApiImpl.kt
│   │       │   │   └── model/
│   │       │   │       └── AuthResponse.kt
│   │       │   └── cache/
│   │       │       ├── SessionCache.kt
│   │       │       └── SessionDao.kt
│   │       └── build.gradle.kts (depends on auth-api)
│   │
│   ├── feature-scrobble/
│   │   ├── scrobble-api/
│   │   │   ├── src/main/java/.../scrobble/api/
│   │   │   │   ├── model/
│   │   │   │   │   ├── PlayedSong.kt
│   │   │   │   │   ├── ScrobbledSong.kt
│   │   │   │   │   └── ScrobbleQueue.kt
│   │   │   │   ├── repository/
│   │   │   │   │   ├── ScrobbleRepository.kt
│   │   │   │   │   └── SongQueueRepository.kt
│   │   │   │   ├── usecase/
│   │   │   │   │   ├── ScrobbleSongsUseCase.kt
│   │   │   │   │   ├── SavePlayedSongUseCase.kt
│   │   │   │   │   ├── SendNowPlayingUseCase.kt
│   │   │   │   │   └── GetQueuedSongsUseCase.kt
│   │   │   │   └── error/
│   │   │   │       └── ScrobbleError.kt
│   │   │   └── build.gradle.kts
│   │   │
│   │   └── scrobble-impl/
│   │       ├── src/main/java/.../scrobble/impl/
│   │       │   ├── di/
│   │       │   ├── repository/
│   │       │   ├── usecase/
│   │       │   ├── remote/
│   │       │   │   └── LastFmScrobbleApi.kt
│   │       │   ├── cache/
│   │       │   │   ├── SongQueueDao.kt
│   │       │   │   └── SongEntity.kt
│   │       │   └── worker/
│   │       │       └── ScrobbleWorker.kt (WorkManager)
│   │       └── build.gradle.kts
│   │
│   ├── feature-songs/
│   │   ├── songs-api/
│   │   │   ├── src/main/java/.../songs/api/
│   │   │   │   ├── model/
│   │   │   │   │   ├── Song.kt
│   │   │   │   │   ├── SongDetails.kt
│   │   │   │   │   └── SongInfo.kt
│   │   │   │   ├── repository/
│   │   │   │   │   └── SongsRepository.kt
│   │   │   │   ├── usecase/
│   │   │   │   │   ├── GetPlayedSongsUseCase.kt
│   │   │   │   │   ├── GetScrobbledSongsUseCase.kt
│   │   │   │   │   ├── GetSongDetailsUseCase.kt
│   │   │   │   │   └── DeleteSongUseCase.kt
│   │   │   │   └── error/
│   │   │   │       └── SongsError.kt
│   │   │   └── build.gradle.kts
│   │   │
│   │   └── songs-impl/
│   │       └── build.gradle.kts
│   │
│   ├── feature-config/
│   │   ├── config-api/
│   │   │   ├── src/main/java/.../config/api/
│   │   │   │   ├── model/
│   │   │   │   │   └── UserConfiguration.kt
│   │   │   │   ├── repository/
│   │   │   │   │   └── ConfigRepository.kt
│   │   │   │   ├── usecase/
│   │   │   │   │   ├── GetConfigUseCase.kt
│   │   │   │   │   └── SaveConfigUseCase.kt
│   │   │   │   └── error/
│   │   │   │       └── ConfigError.kt
│   │   │   └── build.gradle.kts
│   │   │
│   │   └── config-impl/
│   │       └── build.gradle.kts
│   │
│   └── feature-detection/
│       ├── detection-api/
│       │   ├── src/main/java/.../detection/api/
│       │   │   ├── model/
│       │   │   │   └── SongMetadata.kt
│       │   │   ├── service/
│       │   │   │   └── SongDetectionService.kt
│       │   │   └── error/
│       │   │       └── DetectionError.kt
│       │   └── build.gradle.kts
│       │
│       └── detection-impl/
│           ├── src/main/java/.../detection/impl/
│           │   ├── receiver/
│           │   │   └── SpotifyReceiver.kt
│           │   ├── service/
│           │   │   └── DetectionServiceImpl.kt
│           │   └── di/
│           └── build.gradle.kts
│
└── build.gradle.kts (root)
```

### Module Dependencies

```
app
 ├─> :core-api
 ├─> :core-impl
 ├─> :feature-auth-api
 ├─> :feature-auth-impl
 ├─> :feature-scrobble-api
 ├─> :feature-scrobble-impl
 ├─> :feature-songs-api
 ├─> :feature-songs-impl
 ├─> :feature-config-api
 ├─> :feature-config-impl
 ├─> :feature-detection-api
 └─> :feature-detection-impl

feature-X-impl
 ├─> :feature-X-api
 └─> :core-api

feature-X-api
 └─> :core-api (for base error types)

core-impl
 └─> :core-api
```

### Gradle Configuration

**Root `build.gradle.kts`:**
```kotlin
buildscript {
    ext {
        kotlin_version = "1.9.22"
        hilt_version = "2.48"
        arrow_version = "1.2.1"
        coroutines_version = "1.7.3"
    }
}

plugins {
    id("com.android.application") version "8.1.2" apply false
    id("com.android.library") version "8.1.2" apply false
    id("org.jetbrains.kotlin.android") version "1.9.22" apply false
    id("com.google.dagger.hilt.android") version "2.48" apply false
    id("com.google.devtools.ksp") version "1.9.22-1.0.16" apply false
}
```

**Feature API module `build.gradle.kts` example:**
```kotlin
plugins {
    id("com.android.library")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.github.niltsiar.ultimatescrobbler.feature.auth.api"
    compileSdk = 34
}

dependencies {
    implementation(project(":core:core-api"))

    // Arrow for Either
    implementation("io.arrow-kt:arrow-core:1.2.1")

    // Kotlin
    implementation("org.jetbrains.kotlin:kotlin-stdlib:1.9.22")

    // Minimal dependencies - API modules should be lightweight
}
```

**Feature Impl module `build.gradle.kts` example:**
```kotlin
plugins {
    id("com.android.library")
    id("org.jetbrains.kotlin.android")
    id("com.google.dagger.hilt.android")
    id("com.google.devtools.ksp")
}

android {
    namespace = "com.github.niltsiar.ultimatescrobbler.feature.auth.impl"
    compileSdk = 34
}

dependencies {
    implementation(project(":core:core-api"))
    implementation(project(":core:core-impl"))
    implementation(project(":features:feature-auth:auth-api"))

    // Hilt
    implementation("com.google.dagger:hilt-android:2.48")
    ksp("com.google.dagger:hilt-compiler:2.48")

    // Arrow
    implementation("io.arrow-kt:arrow-core:1.2.1")
    implementation("io.arrow-kt:arrow-fx-coroutines:1.2.1")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")

    // Room (if needed)
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    ksp("androidx.room:room-compiler:2.6.1")

    // Retrofit (if needed)
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-moshi:2.9.0")

    // Testing
    testImplementation("junit:junit:4.13.2")
    testImplementation("io.kotest:kotest-assertions-core:5.8.0")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
    testImplementation("io.mockk:mockk:1.13.8")
}
```

---

## Typed Error Hierarchy

### Core Error Types

**Location**: `:core-api/src/main/java/.../core/error/DomainError.kt`

```kotlin
package com.github.niltsiar.ultimatescrobbler.core.error

/**
 * Base interface for all domain errors.
 * All features should extend this for feature-specific errors.
 */
sealed interface DomainError {
    val message: String
    val cause: Throwable?
        get() = null
}

/**
 * Network-related errors
 */
sealed interface NetworkError : DomainError {
    data class ConnectionError(
        override val message: String = "No internet connection",
        override val cause: Throwable? = null
    ) : NetworkError

    data class TimeoutError(
        override val message: String = "Request timed out",
        override val cause: Throwable? = null
    ) : NetworkError

    data class ServerError(
        val code: Int,
        override val message: String,
        override val cause: Throwable? = null
    ) : NetworkError

    data class UnknownError(
        override val message: String = "Unknown network error",
        override val cause: Throwable? = null
    ) : NetworkError
}

/**
 * Local storage errors
 */
sealed interface CacheError : DomainError {
    data class ReadError(
        override val message: String = "Failed to read from cache",
        override val cause: Throwable? = null
    ) : CacheError

    data class WriteError(
        override val message: String = "Failed to write to cache",
        override val cause: Throwable? = null
    ) : CacheError

    data class NotFoundError(
        val key: String,
        override val message: String = "Item not found in cache: $key"
    ) : CacheError
}

/**
 * Validation errors
 */
sealed interface ValidationError : DomainError {
    data class InvalidInput(
        val field: String,
        override val message: String
    ) : ValidationError

    data class MissingRequiredField(
        val field: String,
        override val message: String = "Required field missing: $field"
    ) : ValidationError
}
```

### Feature-Specific Error Types

**Location**: `:features:feature-auth:auth-api/src/main/java/.../auth/api/error/AuthError.kt`

```kotlin
package com.github.niltsiar.ultimatescrobbler.feature.auth.api.error

import com.github.niltsiar.ultimatescrobbler.core.error.DomainError

/**
 * Authentication and authorization errors
 */
sealed interface AuthError : DomainError {

    /**
     * Invalid username or password
     */
    data object InvalidCredentials : AuthError {
        override val message: String = "Invalid username or password"
    }

    /**
     * Session has expired and needs refresh
     */
    data class SessionExpired(
        override val message: String = "Your session has expired. Please login again."
    ) : AuthError

    /**
     * User is not authenticated
     */
    data object NotAuthenticated : AuthError {
        override val message: String = "You are not logged in"
    }

    /**
     * Account is locked or suspended
     */
    data class AccountLocked(
        val reason: String,
        override val message: String = "Account locked: $reason"
    ) : AuthError

    /**
     * API key is invalid or missing
     */
    data object InvalidApiKey : AuthError {
        override val message: String = "Invalid API key"
    }
}
```

**Location**: `:features:feature-scrobble:scrobble-api/src/main/java/.../scrobble/api/error/ScrobbleError.kt`

```kotlin
package com.github.niltsiar.ultimatescrobbler.feature.scrobble.api.error

import com.github.niltsiar.ultimatescrobbler.core.error.DomainError

/**
 * Scrobbling-specific errors
 */
sealed interface ScrobbleError : DomainError {

    /**
     * Last.fm API rate limit exceeded
     */
    data class RateLimitExceeded(
        val retryAfterSeconds: Long,
        override val message: String = "Rate limit exceeded. Retry after $retryAfterSeconds seconds."
    ) : ScrobbleError

    /**
     * Song was rejected by Last.fm
     */
    data class SongRejected(
        val reason: String,
        override val message: String = "Song rejected: $reason"
    ) : ScrobbleError

    /**
     * Invalid song metadata
     */
    data class InvalidSongData(
        val field: String,
        override val message: String = "Invalid song data: $field"
    ) : ScrobbleError

    /**
     * Queue is full
     */
    data class QueueFull(
        val maxSize: Int,
        override val message: String = "Scrobble queue is full (max: $maxSize)"
    ) : ScrobbleError

    /**
     * Song already scrobbled
     */
    data class AlreadyScrobbled(
        val songId: String,
        override val message: String = "Song already scrobbled: $songId"
    ) : ScrobbleError
}
```

### Error Mapping Utilities

**Location**: `:core-impl/src/main/java/.../core/error/ErrorMapper.kt`

```kotlin
package com.github.niltsiar.ultimatescrobbler.core.error

import arrow.core.Either
import arrow.core.left
import arrow.core.right
import retrofit2.HttpException
import java.io.IOException
import java.net.SocketTimeoutException

/**
 * Maps exceptions to typed domain errors
 */
object ErrorMapper {

    fun mapNetworkException(throwable: Throwable): NetworkError {
        return when (throwable) {
            is SocketTimeoutException -> NetworkError.TimeoutError(cause = throwable)
            is IOException -> NetworkError.ConnectionError(
                message = throwable.message ?: "Connection error",
                cause = throwable
            )
            is HttpException -> NetworkError.ServerError(
                code = throwable.code(),
                message = throwable.message(),
                cause = throwable
            )
            else -> NetworkError.UnknownError(
                message = throwable.message ?: "Unknown error",
                cause = throwable
            )
        }
    }

    fun mapCacheException(throwable: Throwable): CacheError {
        return CacheError.ReadError(
            message = throwable.message ?: "Cache error",
            cause = throwable
        )
    }
}

/**
 * Extension to safely execute network calls
 */
suspend inline fun <T> safeNetworkCall(
    crossinline call: suspend () -> T
): Either<NetworkError, T> {
    return try {
        call().right()
    } catch (e: Exception) {
        ErrorMapper.mapNetworkException(e).left()
    }
}

/**
 * Extension to safely execute cache operations
 */
suspend inline fun <T> safeCacheCall(
    crossinline call: suspend () -> T
): Either<CacheError, T> {
    return try {
        call().right()
    } catch (e: Exception) {
        ErrorMapper.mapCacheException(e).left()
    }
}
```

---

## Either and Raise Pattern

### Introduction to Either

`Either<A, B>` is a type that represents a value that can be one of two types:
- `Either.Left<A>` - typically represents an error/failure
- `Either.Right<B>` - typically represents a success value

In our architecture:
- `Either<DomainError, Success>` - Left is error, Right is success

### The raise DSL

Arrow's `raise` DSL simplifies working with Either by:
1. **Automatic unwrapping**: `bind()` automatically extracts the Right value or short-circuits with Left
2. **Early returns**: `raise(error)` immediately returns a Left value
3. **Composition**: Multiple Either operations compose cleanly

### Basic Usage Examples

#### Simple Either Return

```kotlin
package com.github.niltsiar.ultimatescrobbler.feature.config.impl.usecase

import arrow.core.Either
import arrow.core.raise.either
import com.github.niltsiar.ultimatescrobbler.feature.config.api.model.UserConfiguration
import com.github.niltsiar.ultimatescrobbler.feature.config.api.repository.ConfigRepository
import com.github.niltsiar.ultimatescrobbler.feature.config.api.usecase.GetConfigUseCase
import com.github.niltsiar.ultimatescrobbler.core.error.DomainError
import javax.inject.Inject

class GetConfigUseCaseImpl @Inject constructor(
    private val repository: ConfigRepository
) : GetConfigUseCase {

    override suspend fun execute(): Either<DomainError, UserConfiguration> {
        return repository.getConfig()
    }
}
```

#### Using raise for Early Returns

```kotlin
package com.github.niltsiar.ultimatescrobbler.feature.auth.impl.usecase

import arrow.core.Either
import arrow.core.raise.either
import arrow.core.raise.ensure
import com.github.niltsiar.ultimatescrobbler.feature.auth.api.error.AuthError
import com.github.niltsiar.ultimatescrobbler.feature.auth.api.model.Credentials
import com.github.niltsiar.ultimatescrobbler.feature.auth.api.model.Session
import com.github.niltsiar.ultimatescrobbler.feature.auth.api.repository.AuthRepository
import com.github.niltsiar.ultimatescrobbler.feature.auth.api.usecase.LoginUseCase
import com.github.niltsiar.ultimatescrobbler.core.error.ValidationError
import javax.inject.Inject

class LoginUseCaseImpl @Inject constructor(
    private val repository: AuthRepository
) : LoginUseCase {

    override suspend fun execute(credentials: Credentials): Either<AuthError, Session> = either {
        // Validation with ensure - raises ValidationError if condition fails
        ensure(credentials.username.isNotBlank()) {
            ValidationError.MissingRequiredField("username")
        }
        ensure(credentials.password.isNotBlank()) {
            ValidationError.MissingRequiredField("password")
        }

        // Call repository - bind() automatically unwraps or short-circuits
        val session = repository.login(credentials).bind()

        // Additional validation after API call
        ensure(session.token.isNotEmpty()) {
            AuthError.InvalidApiKey
        }

        // Return success value
        session
    }
}
```

#### Composing Multiple Either Operations

```kotlin
package com.github.niltsiar.ultimatescrobbler.feature.scrobble.impl.usecase

import arrow.core.Either
import arrow.core.raise.either
import arrow.core.raise.ensure
import com.github.niltsiar.ultimatescrobbler.feature.scrobble.api.error.ScrobbleError
import com.github.niltsiar.ultimatescrobbler.feature.scrobble.api.model.PlayedSong
import com.github.niltsiar.ultimatescrobbler.feature.scrobble.api.model.ScrobbledSong
import com.github.niltsiar.ultimatescrobbler.feature.scrobble.api.repository.ScrobbleRepository
import com.github.niltsiar.ultimatescrobbler.feature.scrobble.api.usecase.ScrobbleSongsUseCase
import com.github.niltsiar.ultimatescrobbler.feature.config.api.repository.ConfigRepository
import com.github.niltsiar.ultimatescrobbler.feature.auth.api.repository.AuthRepository
import javax.inject.Inject

class ScrobbleSongsUseCaseImpl @Inject constructor(
    private val scrobbleRepository: ScrobbleRepository,
    private val configRepository: ConfigRepository,
    private val authRepository: AuthRepository
) : ScrobbleSongsUseCase {

    override suspend fun execute(songs: List<PlayedSong>): Either<ScrobbleError, List<ScrobbledSong>> = either {
        // Validate input
        ensure(songs.isNotEmpty()) {
            ScrobbleError.InvalidSongData("Songs list is empty")
        }

        // Get session token - bind() unwraps or short-circuits
        val session = authRepository.getCurrentSession().bind()

        // Get user config
        val config = configRepository.getConfig().bind()

        // Check queue size
        val queueSize = scrobbleRepository.getQueueSize().bind()
        ensure(queueSize + songs.size <= config.maxQueueSize) {
            ScrobbleError.QueueFull(config.maxQueueSize)
        }

        // Scrobble songs one by one
        val scrobbledSongs = mutableListOf<ScrobbledSong>()
        for (song in songs) {
            // Validate song
            ensure(song.artistName.isNotBlank()) {
                ScrobbleError.InvalidSongData("Artist name is required")
            }
            ensure(song.trackName.isNotBlank()) {
                ScrobbleError.InvalidSongData("Track name is required")
            }

            // Scrobble
            val scrobbled = scrobbleRepository.scrobble(song, session.token).bind()
            scrobbledSongs.add(scrobbled)
        }

        // Return all scrobbled songs
        scrobbledSongs
    }
}
```

#### Transforming Either Values

```kotlin
package com.github.niltsiar.ultimatescrobbler.feature.songs.impl.usecase

import arrow.core.Either
import arrow.core.raise.either
import com.github.niltsiar.ultimatescrobbler.feature.songs.api.error.SongsError
import com.github.niltsiar.ultimatescrobbler.feature.songs.api.model.Song
import com.github.niltsiar.ultimatescrobbler.feature.songs.api.model.SongDetails
import com.github.niltsiar.ultimatescrobbler.feature.songs.api.repository.SongsRepository
import com.github.niltsiar.ultimatescrobbler.feature.songs.api.usecase.GetSongDetailsUseCase
import javax.inject.Inject

class GetSongDetailsUseCaseImpl @Inject constructor(
    private val repository: SongsRepository
) : GetSongDetailsUseCase {

    override suspend fun execute(songId: String): Either<SongsError, SongDetails> = either {
        // Get basic song info
        val song = repository.getSong(songId).bind()

        // Get additional details from Last.fm
        val additionalInfo = repository.fetchAdditionalInfo(song).bind()

        // Combine into detailed view
        SongDetails(
            id = song.id,
            trackName = song.trackName,
            artistName = song.artistName,
            albumName = song.albumName,
            albumArt = additionalInfo.albumArt,
            tags = additionalInfo.tags,
            wiki = additionalInfo.wiki,
            playCount = additionalInfo.playCount,
            listeners = additionalInfo.listeners
        )
    }
}
```

#### Using fold() for Handling Results

```kotlin
// In ViewModel
class ConfigViewModel @Inject constructor(
    private val getConfigUseCase: GetConfigUseCase,
    private val saveConfigUseCase: SaveConfigUseCase
) : ViewModel() {

    private val _viewState = MutableStateFlow<ConfigViewState>(ConfigViewState.Loading)
    val viewState: StateFlow<ConfigViewState> = _viewState.asStateFlow()

    fun loadConfig() {
        viewModelScope.launch {
            _viewState.value = ConfigViewState.Loading

            getConfigUseCase.execute().fold(
                ifLeft = { error ->
                    // Handle error
                    _viewState.value = ConfigViewState.Error(error.message)
                },
                ifRight = { config ->
                    // Handle success
                    _viewState.value = ConfigViewState.Success(config)
                }
            )
        }
    }

    fun saveConfig(config: UserConfiguration) {
        viewModelScope.launch {
            saveConfigUseCase.execute(config).fold(
                ifLeft = { error ->
                    _viewState.value = ConfigViewState.Error(error.message)
                },
                ifRight = {
                    // Success - reload config
                    loadConfig()
                }
            )
        }
    }
}
```

### Advanced Patterns

#### Parallel Either Operations

```kotlin
import arrow.core.Either
import arrow.core.raise.either
import kotlinx.coroutines.async
import kotlinx.coroutines.coroutineScope

suspend fun loadDashboardData(): Either<DomainError, DashboardData> = either {
    coroutineScope {
        // Launch parallel operations
        val userDeferred = async { userRepository.getUser().bind() }
        val songsDeferred = async { songsRepository.getRecentSongs().bind() }
        val statsDeferred = async { statsRepository.getStats().bind() }

        // Wait for all results
        DashboardData(
            user = userDeferred.await(),
            recentSongs = songsDeferred.await(),
            stats = statsDeferred.await()
        )
    }
}
```

#### Recovering from Errors

```kotlin
import arrow.core.Either
import arrow.core.getOrElse

suspend fun getSongWithFallback(songId: String): Song {
    return repository.getSong(songId)
        .getOrElse {
            // Fallback to default song
            Song.empty()
        }
}

// Or with custom error handling
suspend fun getSongOrCache(songId: String): Either<SongsError, Song> {
    return repository.getSong(songId)
        .mapLeft { error ->
            // Try cache if network fails
            if (error is NetworkError) {
                return cacheRepository.getSong(songId)
            }
            error
        }
}
```

---

## Migration Phases

### Overview

The migration will be executed in **5 phases** over approximately **13-16 weeks**:

1. **Phase 1**: Foundation (2-3 weeks)
2. **Phase 2**: Core Module Restructuring (2 weeks)
3. **Phase 3**: First Feature Migration - Configuration (2 weeks)
4. **Phase 4**: Remaining Features (6-8 weeks)
5. **Phase 5**: Cleanup and Optimization (1 week)

### Phase 1: Foundation (2-3 weeks)

**Goal**: Set up Kotlin, Arrow, and Coroutines infrastructure alongside existing Java/RxJava code.

#### Tasks

1. **Add Kotlin Support**
   - Update `build.gradle` files to support Kotlin
   - Configure Kotlin compiler options
   - Set up Kotlin coding standards

2. **Add Dependencies**
   ```kotlin
   // Root build.gradle.kts
   dependencies {
       implementation("org.jetbrains.kotlin:kotlin-stdlib:1.9.22")
       implementation("io.arrow-kt:arrow-core:1.2.1")
       implementation("io.arrow-kt:arrow-fx-coroutines:1.2.1")
       implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
       implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")

       // Testing
       testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
       testImplementation("io.kotest:kotest-assertions-core:5.8.0")
       testImplementation("io.mockk:mockk:1.13.8")
   }
   ```

3. **Create Error Hierarchy**
   - Create `:core-api` module
   - Define `DomainError` sealed interface
   - Define core error types (NetworkError, CacheError, ValidationError)
   - Create `ErrorMapper` utilities

4. **Set Up Testing Infrastructure**
   - Add test dependencies (Kotest, MockK, Coroutines Test)
   - Create test utilities for Either assertions
   - Set up test conventions

5. **Developer Training**
   - Kotlin fundamentals workshop
   - Coroutines deep dive
   - Arrow Either and raise DSL training
   - Code review guidelines

#### Deliverables

- ✅ Kotlin support enabled
- ✅ Arrow and Coroutines dependencies added
- ✅ `:core-api` module created with error hierarchy
- ✅ Testing infrastructure set up
- ✅ Team trained on new technologies
- ✅ Existing Java/RxJava code still working

#### Success Criteria

- All builds pass
- Existing tests pass
- No regression in app functionality
- Team comfortable with Kotlin and Arrow basics

---

### Phase 2: Core Module Restructuring (2 weeks)

**Goal**: Extract core functionality into reusable modules.

#### Tasks

1. **Create `:core-impl` Module**
   - Extract network setup (OkHttp, Retrofit)
   - Extract database setup (migrate from Schematic to Room)
   - Extract SharedPreferences utilities
   - Convert to Kotlin

2. **Database Migration**
   - Replace Schematic with Room
   - Define Room entities
   - Create DAOs
   - Write migration scripts from old ContentProvider

3. **Network Module**
   - Convert `NetworkModule` to Kotlin
   - Update interceptors for Kotlin
   - Keep Retrofit configuration

4. **Migrate to Hilt**
   - Add Hilt dependencies
   - Create `@HiltAndroidApp` application class
   - Convert `ApplicationComponent` to Hilt
   - Keep Dagger modules as reference during transition

5. **Create Base Classes**
   - Base Use Case interface
   ```kotlin
   interface UseCase<in Params, out Result> {
       suspend fun execute(params: Params): Either<DomainError, Result>
   }
   ```
   - Base Repository interfaces
   - Base ViewModels with StateFlow

#### Deliverables

- ✅ `:core-api` and `:core-impl` modules complete
- ✅ Room database replacing Schematic
- ✅ Hilt DI set up
- ✅ Base classes created
- ✅ Utilities migrated to Kotlin

#### Success Criteria

- App still functions correctly
- Database migration successful (no data loss)
- Hilt injection working
- Core modules compile independently

---

### Phase 3: First Feature Migration - Configuration (2 weeks)

**Goal**: Migrate the simplest feature to establish patterns and templates.

**Why Configuration First?**
- Minimal dependencies
- Simple use cases
- No complex business logic
- Good learning opportunity

#### Tasks

1. **Create Feature Modules**
   - Create `:features:feature-config:config-api`
   - Create `:features:feature-config:config-impl`

2. **Define API Module**
   ```kotlin
   // config-api/model/UserConfiguration.kt
   data class UserConfiguration(
       val username: String,
       val numberOfSongsPerBatch: Int,
       val sendNowPlaying: Boolean
   )

   // config-api/repository/ConfigRepository.kt
   interface ConfigRepository {
       suspend fun getConfig(): Either<ConfigError, UserConfiguration>
       suspend fun saveConfig(config: UserConfiguration): Either<ConfigError, Unit>
   }

   // config-api/usecase/GetConfigUseCase.kt
   interface GetConfigUseCase {
       suspend fun execute(): Either<ConfigError, UserConfiguration>
   }

   // config-api/error/ConfigError.kt
   sealed interface ConfigError : DomainError {
       data object NotFound : ConfigError
       data class InvalidConfig(override val message: String) : ConfigError
   }
   ```

3. **Implement Feature**
   ```kotlin
   // config-impl/repository/ConfigRepositoryImpl.kt
   class ConfigRepositoryImpl @Inject constructor(
       private val preferencesManager: PreferencesManager
   ) : ConfigRepository {

       override suspend fun getConfig(): Either<ConfigError, UserConfiguration> = either {
           safeCacheCall {
               preferencesManager.getConfig()
           }.mapLeft { ConfigError.NotFound }.bind()
       }

       override suspend fun saveConfig(config: UserConfiguration): Either<ConfigError, Unit> = either {
           safeCacheCall {
               preferencesManager.saveConfig(config)
           }.mapLeft { error ->
               ConfigError.InvalidConfig(error.message)
           }.bind()
       }
   }

   // config-impl/usecase/GetConfigUseCaseImpl.kt
   class GetConfigUseCaseImpl @Inject constructor(
       private val repository: ConfigRepository
   ) : GetConfigUseCase {
       override suspend fun execute(): Either<ConfigError, UserConfiguration> {
           return repository.getConfig()
       }
   }
   ```

4. **Update UI Layer**
   ```kotlin
   // app/ui/configuration/ConfigViewModel.kt
   @HiltViewModel
   class ConfigViewModel @Inject constructor(
       private val getConfigUseCase: GetConfigUseCase,
       private val saveConfigUseCase: SaveConfigUseCase
   ) : ViewModel() {

       private val _viewState = MutableStateFlow<ConfigViewState>(ConfigViewState.Loading)
       val viewState: StateFlow<ConfigViewState> = _viewState.asStateFlow()

       fun loadConfig() {
           viewModelScope.launch {
               getConfigUseCase.execute().fold(
                   ifLeft = { error -> _viewState.value = ConfigViewState.Error(error.message) },
                   ifRight = { config -> _viewState.value = ConfigViewState.Success(config) }
               )
           }
       }
   }
   ```

5. **Write Tests**
   - Unit tests for use cases
   - Repository tests
   - ViewModel tests

6. **Create Migration Template**
   - Document the process
   - Create module templates
   - Establish code review checklist

#### Deliverables

- ✅ Configuration feature fully migrated
- ✅ All tests passing
- ✅ Migration template created
- ✅ No regression in configuration functionality

#### Success Criteria

- Configuration feature works identically to before
- Code is cleaner and more testable
- Team understands the migration pattern
- Ready to scale to other features

---

### Phase 4: Remaining Features (6-8 weeks, ~2 weeks per feature)

**Goal**: Migrate all remaining features to the new architecture.

#### Feature Migration Order

1. **Feature: Authentication** (2 weeks)
   - Simplest after configuration
   - Required by other features
   - Clear error cases

2. **Feature: Song Detection** (2 weeks)
   - Migrate SpotifyReceiver
   - Convert RxRelay to StateFlow
   - Debouncing with Coroutines Flow

3. **Feature: Songs** (2 weeks)
   - Viewing played songs
   - Viewing scrobbled songs
   - Song details

4. **Feature: Scrobble** (2-3 weeks)
   - Most complex feature
   - Queue management
   - Background processing
   - Rate limiting

#### Per-Feature Tasks

For each feature:

1. **Create Modules**
   - `feature-X-api` module
   - `feature-X-impl` module

2. **Define API**
   - Domain models
   - Repository interfaces
   - Use case interfaces
   - Feature-specific errors

3. **Implement**
   - Repository implementations
   - Use case implementations
   - Data sources (Remote/Cache)
   - Mappers

4. **Update UI**
   - Convert ViewModels to use new use cases
   - Update to StateFlow
   - Handle typed errors in UI

5. **Background Work**
   - Replace Firebase JobDispatcher with WorkManager
   - Convert to Coroutines

6. **Test**
   - Unit tests
   - Integration tests
   - UI tests

7. **Review & Iterate**
   - Code review
   - Performance testing
   - Bug fixes

#### Authentication Feature Details

**API Module**:
```kotlin
// auth-api/model/
data class Credentials(val username: String, val password: String)
data class Session(val token: String, val expiresAt: Instant)

// auth-api/repository/
interface AuthRepository {
    suspend fun login(credentials: Credentials): Either<AuthError, Session>
    suspend fun logout(): Either<AuthError, Unit>
    suspend fun getCurrentSession(): Either<AuthError, Session>
    suspend fun refreshSession(): Either<AuthError, Session>
}

// auth-api/usecase/
interface LoginUseCase {
    suspend fun execute(credentials: Credentials): Either<AuthError, Session>
}

interface LogoutUseCase {
    suspend fun execute(): Either<AuthError, Unit>
}

// auth-api/error/
sealed interface AuthError : DomainError {
    data object InvalidCredentials : AuthError
    data object SessionExpired : AuthError
    data object NotAuthenticated : AuthError
}
```

#### Song Detection Feature Details

**Migration from RxRelay to Flow**:

```kotlin
// Old (RxJava)
class SpotifyReceiver : BroadcastReceiver {
    private val playedSongs = PublishRelay.create<PlayedSong>()

    fun getPlayedSongs(): Observable<PlayedSong> = playedSongs
}

// New (Coroutines Flow)
class SpotifyReceiverImpl : BroadcastReceiver, SongDetectionService {
    private val _playedSongs = MutableSharedFlow<PlayedSong>()
    override val playedSongs: SharedFlow<PlayedSong> = _playedSongs.asSharedFlow()

    override fun onReceive(context: Context, intent: Intent) {
        val song = extractSongFromIntent(intent)
        viewModelScope.launch {
            _playedSongs.emit(song)
        }
    }
}

// Debouncing
val debouncedSongs = playedSongs
    .debounce(10_000) // 10 seconds
    .shareIn(
        scope = serviceScope,
        started = SharingStarted.Eagerly
    )
```

#### Scrobble Feature Details (Most Complex)

**Background Work Migration**:

```kotlin
// Old (Firebase JobDispatcher)
class ScrobblePlayedSongsService : JobService {
    override fun onStartJob(job: JobParameters): Boolean {
        // RxJava chain
    }
}

// New (WorkManager with Coroutines)
@HiltWorker
class ScrobbleWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted workerParams: WorkerParameters,
    private val scrobbleSongsUseCase: ScrobbleSongsUseCase,
    private val getQueuedSongsUseCase: GetQueuedSongsUseCase
) : CoroutineWorker(context, workerParams) {

    override suspend fun doWork(): Result {
        return getQueuedSongsUseCase.execute().fold(
            ifLeft = { error ->
                if (error is NetworkError) {
                    Result.retry() // Retry on network errors
                } else {
                    Result.failure()
                }
            },
            ifRight = { songs ->
                scrobbleSongsUseCase.execute(songs).fold(
                    ifLeft = { Result.failure() },
                    ifRight = { Result.success() }
                )
            }
        )
    }
}

// Scheduling
fun scheduleScrobble() {
    val constraints = Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .build()

    val request = OneTimeWorkRequestBuilder<ScrobbleWorker>()
        .setConstraints(constraints)
        .setBackoffCriteria(
            BackoffPolicy.LINEAR,
            OneTimeWorkRequest.MIN_BACKOFF_MILLIS,
            TimeUnit.MILLISECONDS
        )
        .build()

    WorkManager.getInstance(context).enqueue(request)
}
```

#### Deliverables (per feature)

- ✅ Feature modules created (API + Impl)
- ✅ All use cases migrated
- ✅ All repositories migrated
- ✅ UI updated
- ✅ Tests written and passing
- ✅ Feature works identically to before

#### Success Criteria (overall)

- All 5 features migrated
- Zero critical bugs
- Test coverage maintained or improved
- Performance same or better
- Code review approved

---

### Phase 5: Cleanup and Optimization (1 week)

**Goal**: Remove old code, optimize, and finalize the migration.

#### Tasks

1. **Remove Old Modules**
   - Delete `:domain` module
   - Delete `:data` module
   - Delete `:remote` module
   - Delete `:cache` module

2. **Remove Old Dependencies**
   ```kotlin
   // Remove from build.gradle
   // - RxJava2
   // - RxAndroid
   // - RxBinding
   // - RxRelay
   // - Schematic
   // - Old Dagger (keep Hilt)
   ```

3. **Code Cleanup**
   - Remove all Java files
   - Remove unused imports
   - Format all Kotlin files
   - Update documentation

4. **Performance Optimization**
   - Profile app startup time
   - Optimize database queries
   - Optimize network calls
   - Reduce APK size

5. **Update Documentation**
   - Update Architecture.md
   - Update README.md
   - Create migration retrospective
   - Document new patterns

6. **Final Testing**
   - Full regression testing
   - Performance testing
   - Memory leak testing
   - Battery usage testing

#### Deliverables

- ✅ Old code removed
- ✅ Dependencies cleaned up
- ✅ Documentation updated
- ✅ Performance optimized
- ✅ Final testing complete

#### Success Criteria

- 100% Kotlin codebase
- No RxJava dependencies
- All features working
- Performance metrics acceptable
- Team trained and confident

---

## Technology Stack Changes

### Before → After Comparison

| Category | Before | After |
|----------|--------|-------|
| **Language** | Java 8 | Kotlin 1.9+ |
| **Architecture** | Horizontal layers | Vertical features + API/Impl |
| **Async** | RxJava2 | Kotlin Coroutines + Flow |
| **Error Handling** | Exceptions | Arrow Either + raise DSL |
| **DI** | Dagger 2.14.1 | Hilt 2.48 |
| **Database** | Schematic | Room 2.6+ |
| **Background Work** | Firebase JobDispatcher | WorkManager 2.9+ |
| **State Management** | RxRelay (BehaviorRelay, PublishRelay) | StateFlow, SharedFlow |
| **Testing** | JUnit 4 + Mockito | JUnit 4 + Kotest + MockK |

### New Dependencies

```kotlin
// build.gradle.kts (app module)

dependencies {
    // Kotlin
    implementation("org.jetbrains.kotlin:kotlin-stdlib:1.9.22")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")

    // Arrow for Either
    implementation("io.arrow-kt:arrow-core:1.2.1")
    implementation("io.arrow-kt:arrow-fx-coroutines:1.2.1")

    // Hilt for DI
    implementation("com.google.dagger:hilt-android:2.48")
    ksp("com.google.dagger:hilt-compiler:2.48")

    // Room for database
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    ksp("androidx.room:room-compiler:2.6.1")

    // WorkManager for background
    implementation("androidx.work:work-runtime-ktx:2.9.0")

    // Lifecycle with Coroutines
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.6.2")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.6.2")

    // Retrofit (keep, add coroutines adapter)
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-moshi:2.9.0")

    // Testing
    testImplementation("junit:junit:4.13.2")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
    testImplementation("io.kotest:kotest-assertions-core:5.8.0")
    testImplementation("io.kotest:kotest-assertions-arrow:5.8.0")
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("app.cash.turbine:turbine:1.0.0") // For testing Flow
}
```

### Removed Dependencies

```kotlin
// Remove these from build.gradle
- io.reactivex.rxjava2:rxjava
- io.reactivex.rxjava2:rxandroid
- com.jakewharton.rxbinding2:rxbinding
- com.jakewharton.rxrelay2:rxrelay
- net.simonvt.schematic:schematic
- com.firebase:firebase-jobdispatcher
- com.google.dagger:dagger (keep Hilt which is built on Dagger)
- com.google.dagger:dagger-android
- com.google.dagger:dagger-android-support
```

---

## Code Examples

### Repository Pattern with Either

**Before (RxJava)**:
```java
public class ScrobblerDataRepository implements ScrobblerRepository {

    @Override
    public Single<List<PlayedSong>> getStoredPlayedSongs() {
        return songsCache.getStoredPlayedSongs()
            .flatMapObservable(Observable::fromIterable)
            .map(playedSongMapper::mapFromEntity)
            .toList();
    }

    @Override
    public Observable<ScrobbledSong> scrobblePlayedSongs(List<PlayedSong> playedSongs) {
        List<PlayedSongEntity> entities = new ArrayList<>();
        for (PlayedSong song : playedSongs) {
            entities.add(playedSongMapper.mapToEntity(song));
        }

        return scrobblerRemote.get()
            .scrobblePlayedSongs(entities)
            .map(scrobbledSongMapper::mapFromEntity);
    }
}
```

**After (Kotlin + Either)**:
```kotlin
class ScrobbleRepositoryImpl @Inject constructor(
    private val scrobbleApi: ScrobbleApi,
    private val songQueueDao: SongQueueDao,
    private val mapper: SongMapper
) : ScrobbleRepository {

    override suspend fun getQueuedSongs(): Either<DomainError, List<PlayedSong>> = either {
        safeCacheCall {
            songQueueDao.getAll()
        }.map { entities ->
            entities.map { mapper.toDomain(it) }
        }.bind()
    }

    override suspend fun scrobbleSongs(
        songs: List<PlayedSong>,
        sessionToken: String
    ): Either<ScrobbleError, List<ScrobbledSong>> = either {
        ensure(songs.isNotEmpty()) {
            ScrobbleError.InvalidSongData("Songs list is empty")
        }

        val entities = songs.map { mapper.toEntity(it) }

        safeNetworkCall {
            scrobbleApi.scrobbleSongs(entities, sessionToken)
        }.mapLeft { networkError ->
            // Map network error to scrobble-specific error
            when (networkError) {
                is NetworkError.ServerError ->
                    if (networkError.code == 429) {
                        ScrobbleError.RateLimitExceeded(60)
                    } else {
                        networkError
                    }
                else -> networkError
            }
        }.map { response ->
            response.map { mapper.toScrobbledSong(it) }
        }.bind()
    }
}
```

### Use Case Pattern with raise

**Before (RxJava)**:
```java
public class GetPlayedSongsUseCase extends SingleUseCase<List<PlayedSong>, Void> {

    private final ScrobblerRepository repository;

    @Inject
    public GetPlayedSongsUseCase(
        ScrobblerRepository repository,
        Scheduler executionScheduler,
        Scheduler postExecutionScheduler) {

        super(executionScheduler, postExecutionScheduler);
        this.repository = repository;
    }

    @Override
    protected Single<List<PlayedSong>> buildUseCaseObservable(@Nullable Void param) {
        return repository.getStoredPlayedSongs();
    }
}
```

**After (Kotlin + Either)**:
```kotlin
class GetQueuedSongsUseCaseImpl @Inject constructor(
    private val repository: ScrobbleRepository
) : GetQueuedSongsUseCase {

    override suspend fun execute(): Either<DomainError, List<PlayedSong>> {
        return repository.getQueuedSongs()
    }
}

// More complex example with validation
class ScrobbleSongsUseCaseImpl @Inject constructor(
    private val scrobbleRepository: ScrobbleRepository,
    private val authRepository: AuthRepository
) : ScrobbleSongsUseCase {

    override suspend fun execute(songs: List<PlayedSong>): Either<ScrobbleError, List<ScrobbledSong>> = either {
        // Validate input
        ensure(songs.isNotEmpty()) {
            ScrobbleError.InvalidSongData("Songs list cannot be empty")
        }

        // Get current session
        val session = authRepository.getCurrentSession().mapLeft { authError ->
            ScrobbleError.AuthenticationRequired(authError.message)
        }.bind()

        // Scrobble songs
        scrobbleRepository.scrobbleSongs(songs, session.token).bind()
    }
}
```

### ViewModel with StateFlow

**Before (RxJava + BehaviorRelay)**:
```java
public class PlayedSongsViewModel extends ViewModel {

    private final GetPlayedSongsUseCase getPlayedSongsUseCase;
    private final BehaviorRelay<PlayedSongsViewState> viewStateRelay = BehaviorRelay.create();
    private final CompositeDisposable disposables = new CompositeDisposable();

    @Inject
    public PlayedSongsViewModel(GetPlayedSongsUseCase getPlayedSongsUseCase) {
        this.getPlayedSongsUseCase = getPlayedSongsUseCase;
    }

    public Observable<PlayedSongsViewState> getViewState() {
        return viewStateRelay;
    }

    public void loadSongs() {
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
                    viewStateRelay::accept,
                    error -> viewStateRelay.accept(PlayedSongsViewState.error(error.getMessage()))
                )
        );
    }

    @Override
    protected void onCleared() {
        disposables.clear();
        super.onCleared();
    }
}
```

**After (Kotlin + StateFlow + Either)**:
```kotlin
@HiltViewModel
class PlayedSongsViewModel @Inject constructor(
    private val getQueuedSongsUseCase: GetQueuedSongsUseCase,
    private val deleteSongUseCase: DeleteSongUseCase
) : ViewModel() {

    private val _viewState = MutableStateFlow<PlayedSongsViewState>(PlayedSongsViewState.Loading)
    val viewState: StateFlow<PlayedSongsViewState> = _viewState.asStateFlow()

    init {
        loadSongs()
    }

    fun loadSongs() {
        viewModelScope.launch {
            _viewState.value = PlayedSongsViewState.Loading

            getQueuedSongsUseCase.execute().fold(
                ifLeft = { error ->
                    _viewState.value = PlayedSongsViewState.Error(error.message)
                },
                ifRight = { songs ->
                    _viewState.value = if (songs.isEmpty()) {
                        PlayedSongsViewState.Empty
                    } else {
                        PlayedSongsViewState.Success(songs)
                    }
                }
            )
        }
    }

    fun deleteSong(song: PlayedSong) {
        viewModelScope.launch {
            deleteSongUseCase.execute(song).fold(
                ifLeft = { error ->
                    // Show error snackbar or similar
                    _viewState.value = PlayedSongsViewState.Error(error.message)
                },
                ifRight = {
                    // Reload after deletion
                    loadSongs()
                }
            )
        }
    }
}

sealed interface PlayedSongsViewState {
    data object Loading : PlayedSongsViewState
    data object Empty : PlayedSongsViewState
    data class Success(val songs: List<PlayedSong>) : PlayedSongsViewState
    data class Error(val message: String) : PlayedSongsViewState
}
```

### Fragment/Activity Observing ViewModel

**Before (RxJava)**:
```java
public class PlayedSongsFragment extends Fragment {

    @Inject PlayedSongsViewModelFactory viewModelFactory;
    private PlayedSongsViewModel viewModel;
    private CompositeDisposable disposables = new CompositeDisposable();

    @Override
    public void onResume() {
        super.onResume();

        disposables.add(
            viewModel.getViewState()
                .subscribe(this::render, Timber::e)
        );

        viewModel.loadSongs();
    }

    @Override
    public void onPause() {
        disposables.clear();
        super.onPause();
    }

    private void render(PlayedSongsViewState viewState) {
        // Update UI
    }
}
```

**After (Kotlin + StateFlow)**:
```kotlin
@AndroidEntryPoint
class PlayedSongsFragment : Fragment(R.layout.fragment_played_songs) {

    private val viewModel: PlayedSongsViewModel by viewModels()
    private val binding by viewBinding(FragmentPlayedSongsBinding::bind)

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // Collect StateFlow in lifecycle-aware manner
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.viewState.collect { state ->
                    render(state)
                }
            }
        }
    }

    private fun render(state: PlayedSongsViewState) {
        when (state) {
            PlayedSongsViewState.Loading -> showLoading()
            PlayedSongsViewState.Empty -> showEmpty()
            is PlayedSongsViewState.Success -> showSongs(state.songs)
            is PlayedSongsViewState.Error -> showError(state.message)
        }
    }

    private fun showLoading() {
        binding.progressBar.isVisible = true
        binding.recyclerView.isVisible = false
        binding.emptyView.isVisible = false
    }

    private fun showSongs(songs: List<PlayedSong>) {
        binding.progressBar.isVisible = false
        binding.recyclerView.isVisible = true
        binding.emptyView.isVisible = false
        adapter.submitList(songs)
    }

    private fun showEmpty() {
        binding.progressBar.isVisible = false
        binding.recyclerView.isVisible = false
        binding.emptyView.isVisible = true
    }

    private fun showError(message: String) {
        Snackbar.make(binding.root, message, Snackbar.LENGTH_LONG).show()
    }
}
```

### Background Work with WorkManager

**Before (Firebase JobDispatcher)**:
```java
public class ScrobblePlayedSongsService extends JobService {

    @Inject GetStoredPlayedSongsUseCase getStoredPlayedSongsUseCase;
    @Inject ScrobbleSongsUseCase scrobbleSongsUseCase;

    @Override
    public boolean onStartJob(JobParameters job) {
        AndroidInjection.inject(this);

        getStoredPlayedSongsUseCase.execute(null)
            .flatMapObservable(songs -> scrobbleSongsUseCase.execute(songs))
            .subscribe(
                () -> finishJob(job, false),
                error -> {
                    Timber.e(error);
                    finishJob(job, true);
                }
            );

        return true;
    }

    public static Job createJob(FirebaseJobDispatcher dispatcher) {
        return dispatcher.newJobBuilder()
            .setService(ScrobblePlayedSongsService.class)
            .setConstraints(Constraint.ON_ANY_NETWORK)
            .build();
    }
}
```

**After (WorkManager + Coroutines + Either)**:
```kotlin
@HiltWorker
class ScrobbleWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted workerParams: WorkerParameters,
    private val getQueuedSongsUseCase: GetQueuedSongsUseCase,
    private val scrobbleSongsUseCase: ScrobbleSongsUseCase,
    private val deleteSongsUseCase: DeleteSongsUseCase
) : CoroutineWorker(context, workerParams) {

    override suspend fun doWork(): Result {
        return getQueuedSongsUseCase.execute().fold(
            ifLeft = { error ->
                when (error) {
                    is NetworkError -> Result.retry() // Retry on network errors
                    else -> Result.failure()
                }
            },
            ifRight = { songs ->
                if (songs.isEmpty()) {
                    return Result.success()
                }

                scrobbleSongsUseCase.execute(songs).fold(
                    ifLeft = { error ->
                        when (error) {
                            is ScrobbleError.RateLimitExceeded -> {
                                // Schedule retry after rate limit expires
                                Result.retry()
                            }
                            is NetworkError -> Result.retry()
                            else -> Result.failure()
                        }
                    },
                    ifRight = { scrobbledSongs ->
                        // Delete successfully scrobbled songs
                        deleteSongsUseCase.execute(songs).fold(
                            ifLeft = { Result.failure() },
                            ifRight = { Result.success() }
                        )
                    }
                )
            }
        )
    }
}

// Scheduling the worker
class ScrobbleScheduler @Inject constructor(
    @ApplicationContext private val context: Context
) {
    fun scheduleScrobble() {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()

        val request = OneTimeWorkRequestBuilder<ScrobbleWorker>()
            .setConstraints(constraints)
            .setBackoffCriteria(
                BackoffPolicy.LINEAR,
                WorkRequest.MIN_BACKOFF_MILLIS,
                TimeUnit.MILLISECONDS
            )
            .build()

        WorkManager.getInstance(context).enqueue(request)
    }
}
```

---

## Risks and Mitigation

### Risk 1: Big Bang Migration Failure

**Risk**: Attempting to migrate everything at once leads to broken app and lost productivity.

**Mitigation**:
- ✅ Incremental migration (one feature at a time)
- ✅ Keep old code running alongside new code
- ✅ Feature flags to toggle between implementations
- ✅ Thorough testing after each feature migration

### Risk 2: Team Learning Curve

**Risk**: Team unfamiliar with Kotlin, Coroutines, and Arrow slows down development.

**Mitigation**:
- ✅ Comprehensive training workshops
- ✅ Pair programming during migration
- ✅ Code review guidelines and checklists
- ✅ Migration templates and examples
- ✅ Documentation and reference materials

### Risk 3: Regression Bugs

**Risk**: Migration introduces bugs that weren't in the original code.

**Mitigation**:
- ✅ Maintain or improve test coverage
- ✅ Extensive regression testing
- ✅ QA involvement throughout migration
- ✅ Beta testing with subset of users
- ✅ Rollback plan for each phase

### Risk 4: Performance Degradation

**Risk**: New architecture is slower or uses more memory than old one.

**Mitigation**:
- ✅ Performance benchmarking before migration (baseline)
- ✅ Performance testing after each phase
- ✅ Profiling tools (Android Profiler, LeakCanary)
- ✅ Memory leak detection
- ✅ Battery usage monitoring

### Risk 5: Dependency Incompatibilities

**Risk**: Some libraries don't work well with Kotlin Coroutines or Arrow.

**Mitigation**:
- ✅ Research library compatibility before migration
- ✅ Find Kotlin-friendly alternatives (e.g., Room vs Schematic)
- ✅ Write adapters/wrappers if needed
- ✅ Contribute to open-source libraries if necessary

### Risk 6: Database Migration Issues

**Risk**: Migrating from Schematic ContentProvider to Room loses user data.

**Mitigation**:
- ✅ Write and test migration scripts
- ✅ Backup data before migration
- ✅ Test on multiple devices and Android versions
- ✅ Gradual rollout with monitoring
- ✅ Rollback mechanism if data loss detected

### Risk 7: Scope Creep

**Risk**: Team wants to add new features or refactor more than planned.

**Mitigation**:
- ✅ Strict scope definition for migration
- ✅ "No new features" rule during migration
- ✅ Backlog for post-migration improvements
- ✅ Regular progress reviews
- ✅ Project manager oversight

### Risk 8: Timeline Delays

**Risk**: Migration takes longer than 13-16 weeks estimated.

**Mitigation**:
- ✅ Buffer time in estimates (20% contingency)
- ✅ Track progress weekly
- ✅ Identify blockers early
- ✅ Adjust scope if needed (defer non-critical features)
- ✅ Dedicated migration team (no context switching)

---

## Timeline and Resources

### Estimated Timeline

| Phase | Duration | Start | End |
|-------|----------|-------|-----|
| **Phase 1**: Foundation | 3 weeks | Week 1 | Week 3 |
| **Phase 2**: Core Restructuring | 2 weeks | Week 4 | Week 5 |
| **Phase 3**: Config Feature | 2 weeks | Week 6 | Week 7 |
| **Phase 4**: Remaining Features | 7 weeks | Week 8 | Week 14 |
| - Authentication | 2 weeks | Week 8 | Week 9 |
| - Song Detection | 2 weeks | Week 10 | Week 11 |
| - Songs | 2 weeks | Week 12 | Week 13 |
| - Scrobble | 3 weeks | Week 14 | Week 16 |
| **Phase 5**: Cleanup | 1 week | Week 17 | Week 17 |
| **Total** | **17 weeks** | | |

With 20% buffer: **~20 weeks (5 months)**

### Resource Requirements

#### Team Composition

- **2 Senior Android Developers**: Lead migration, establish patterns
- **2 Mid-level Android Developers**: Implement features under guidance
- **1 QA Engineer**: Testing each phase
- **1 Tech Lead**: Code review, architecture decisions
- **1 Project Manager**: Timeline tracking, stakeholder communication

#### Training Investment

- **Week 1**: Kotlin bootcamp (2 days)
- **Week 2**: Coroutines workshop (1 day)
- **Week 2**: Arrow and Either training (1 day)
- **Ongoing**: Pair programming and code reviews

#### Tools and Infrastructure

- **IDE**: Android Studio latest stable
- **CI/CD**: Jenkins/GitHub Actions with Kotlin support
- **Testing**: Expanded test suite, performance benchmarks
- **Monitoring**: Firebase Crashlytics, Performance Monitoring
- **Version Control**: Git with feature branches per phase

### Progress Tracking

**Weekly Milestones**:
- Week 1-3: Foundation complete, team trained
- Week 5: Core modules migrated
- Week 7: First feature (config) complete
- Week 9: Authentication complete
- Week 11: Song detection complete
- Week 13: Songs feature complete
- Week 16: Scrobble feature complete
- Week 17: Old code removed, migration complete

**Metrics**:
- % of codebase in Kotlin
- % of features migrated
- Test coverage %
- Number of critical bugs
- Performance benchmarks (app startup, memory usage)

---

## Success Criteria

### Technical Criteria

✅ **100% Kotlin Codebase**
- No Java files remaining
- All code follows Kotlin coding standards

✅ **Feature Module Architecture**
- All features in separate modules (API + Impl)
- Clear module dependencies
- No circular dependencies

✅ **Typed Error Handling**
- All functions return `Either<Error, Success>`
- Comprehensive error hierarchy
- Errors are handled in UI

✅ **Coroutines for Async**
- No RxJava dependencies
- All async operations use Coroutines
- Flow for event streams

✅ **Test Coverage**
- Maintain or exceed current test coverage
- Unit tests for all use cases
- Integration tests for repositories
- UI tests for critical paths

✅ **Performance**
- App startup time ≤ current baseline
- Memory usage ≤ current baseline
- Battery usage ≤ current baseline
- No memory leaks

✅ **Zero Critical Bugs**
- No P0/P1 bugs introduced by migration
- All existing features work identically

### Process Criteria

✅ **Team Readiness**
- All team members trained on Kotlin, Coroutines, Arrow
- Team confident in new architecture
- Code review process established

✅ **Documentation**
- Architecture.md updated
- Migration completed retrospective documented
- Code examples and templates created
- API documentation for all modules

✅ **Clean Codebase**
- Old code removed
- Unused dependencies removed
- Code formatted and linted
- No warnings or errors in build

### Business Criteria

✅ **No User-Facing Regressions**
- All features work as before
- No data loss
- No performance degradation

✅ **Maintainability Improved**
- Easier to add new features
- Faster to fix bugs
- Better code organization

✅ **Timeline Met**
- Migration completed within 20 weeks
- No major delays

---

## Appendix

### Glossary

- **Either**: A type representing a value that can be one of two types (Left or Right)
- **raise**: Arrow DSL for early returns in Either computations
- **bind()**: Unwraps an Either.Right value or short-circuits with Either.Left
- **Coroutine**: Kotlin's lightweight thread for async programming
- **Flow**: Kotlin's reactive stream (like RxJava Observable)
- **StateFlow**: Hot Flow that holds state (like RxJava BehaviorRelay)
- **suspend**: Kotlin keyword for coroutine functions
- **Sealed Interface**: Kotlin type for restricted class hierarchies (like enum but more powerful)

### References

- [Kotlin Documentation](https://kotlinlang.org/docs/home.html)
- [Kotlin Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html)
- [Arrow Documentation](https://arrow-kt.io/docs/)
- [Arrow Either Guide](https://arrow-kt.io/docs/apidocs/arrow-core/arrow.core/-either/)
- [Arrow Raise](https://arrow-kt.io/docs/typed-errors/working-with-typed-errors/)
- [Hilt Documentation](https://dagger.dev/hilt/)
- [Room Documentation](https://developer.android.com/training/data-storage/room)
- [WorkManager Guide](https://developer.android.com/topic/libraries/architecture/workmanager)

### Contact

For questions or clarifications about this migration plan, contact:
- **Tech Lead**: [Name]
- **Project Manager**: [Name]

---

**Document Version**: 1.0
**Last Updated**: 2025-11-03
**Status**: Draft - Awaiting Approval
