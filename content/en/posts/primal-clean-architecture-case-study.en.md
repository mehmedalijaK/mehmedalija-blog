---
title: "Diving into Primal: A Clean Architecture Case Study"
date: 2025-09-23T14:30:00+02:00
draft: false
tags: ["Android", "Kotlin", "Clean Architecture", "Nostr", "Primal", "Software Architecture"]
categories: ["Software Architecture"]
description: "A deep dive into the modular structure of the Primal Android app, explaining how it uses Clean Architecture principles to build a scalable Nostr client."
translationKey: "primal-clean-architecture-post"
---

My journey into Clean Architecture began with a simple goal: to deeply understand Primal, the open-source Nostr project I'm contributing to. I chose to write about this project because I genuinely believe it's a prime example of a well-structured application that follows modern development principles.

It's an open-source project that can stand alongside modern social media giants like X (formerly Twitter), Facebook, and YouTube, all while championing the core principle of censorship-resistant communication.

So, let’s begin.

### 1. A Brief Introduction to Clean Architecture Principles

At its core, Primal's architecture is built on layers, each with a single, clear responsibility. Think of it as a set of concentric circles. The fundamental principle is the **Dependency Rule**: outer layers can depend on inner layers, but inner layers must remain completely independent of the outer ones.

This makes perfect sense. An `Entity`—our core data model—shouldn't change just because we switch from one UI library to another or fetch data from a new source. The outer layers, like the UI, are the parts we change most often, while the inner layers, our business logic, should remain stable.

This illustration shows how it works, and we'll explore each layer.

![The Clean Coder Blog by Robert C. Martin (Uncle Bob)](/images/clean-architecture.jpg)

### The Domain Layer: The Core of the Application

As the illustration shows, the `domain` layer is the innermost circle. It depends on nothing. This is the heart of the application, containing all the core business logic, data models, and repository interfaces. Crucially, it has **zero Android dependencies**.

In the Primal project, the `domain` layer is organized into packages like `nostr`, `primal`, and `wallet`. Let’s look at `NostrEvent`, a fundamental model in the `nostr` package:

```kotlin
@Serializable
data class NostrEvent(
    val id: String,
    @SerialName("pubkey") val pubKey: String,
    @SerialName("created_at") val createdAt: Long,
    val kind: Int,
    val tags: List<JsonArray> = emptyList(),
    val content: String,
    val sig: String,
)
```

This data class defines what a Nostr event is. It will always have properties like a `pubKey` and a `createdAt` timestamp. It doesn't matter if we use a different database engine or another technology; this is how a `NostrEvent` is represented throughout the system.

Beyond models, the domain layer defines contracts for how data should be managed through repository interfaces. For example, the `LightningRepository` interface dictates how any part of the app should interact with Lightning invoices:

```kotlin
interface LightningRepository {
    suspend fun getPayRequest(lnUrl: String): Result<PayRequest>

    suspend fun getInvoice(
        callbackUrl: String,
        amountMSats: Long,
        comment: String?,
    ): Result<InvoiceResponse>
}
```

Now, any part of the application that needs to handle invoices knows it must go through this interface and can expect these specific return values. As you can see, there are **no Android dependencies here**.

This is where the magic happens. Because this layer is pure Kotlin, the Primal iOS app can share this exact same `domain` module. We get to have the same core business logic for both Android and iOS while building fully native UIs with Jetpack Compose and SwiftUI, respectively (which is amazing).

Similarly, in the `primal` package, we define domain-specific models like `Stream` and their corresponding repository interfaces:

```kotlin
data class Stream(
    val aTag: String,
    val eventId: String,
    val eventAuthorId: String,
    // ... other properties
)

interface StreamRepository {
    fun observeLiveStreamsByMainHostId(mainHostId: String): Flow<List<Stream>>
    suspend fun findWhoIsLive(mainHostIds: List<String>): Set<String>
    fun observeStream(aTag: String): Flow<Stream?>
    // ...
}
```

To conclude, the `domain` layer is where we define our business logic. It provides a clear contract for our Android and iOS applications, dictating **WHAT** the application does, not **HOW** it does it. This clean separation is the foundation of a robust and maintainable system.

### The Use Case Layer: Where the Magic Happens

We just talked about the `domain` layer—the core business rules that are universal across the Primal ecosystem. Now, let's move one layer out to the **Use Case Layer**, which represents the **application-specific business logic**.

This layer is responsible for orchestrating the flow of data to and from the domain entities. It executes specific actions a user can perform in the Primal app, such as:

- Liking a post
- Bookmarking a note
- Muting a user
- Sending a zap…

Essentially, a use case coordinates all the necessary steps to complete an action. For instance, when you like a post in Primal, the application first performs an **optimistic update** (immediately showing the post as liked in the UI) and then publishes the "like" event to the Nostr network. The use case layer orchestrates this entire process.

In the Primal project, you won't find a dedicated `usecase` module. Instead, this logic is pragmatically integrated within the **repository implementations** in the `data` module. These repositories aren't just simple data containers; they act as the use cases themselves.

Let's look at two examples from `EventInteractionRepositoryImpl.kt`.

**Example 1: Liking a Post**

The `likeEvent` method is a perfect illustration of a use case. It doesn't just send a network request; it manages the complete flow:

```kotlin
override suspend fun likeEvent(
    userId: String,
    eventId: String,
    eventAuthorId: String,
    // ...
) = withContext(dispatcherProvider.io()) {
    val statsUpdater = EventStatsUpdater(...) // Prepares for UI updates

    try {
        // 1. Optimistically update the local database
        statsUpdater.increaseLikeStats()

        // 2. Sign and publish the event to the Nostr network
        primalPublisher.signPublishImportNostrEvent(...)
    } catch (error: NostrPublishException) {
        // 3. Revert the local database state if the network call fails
        statsUpdater.revertStats()
        throw error
    }
    // ...
}
```
This is a classic use case: it orchestrates the local database (`EventStatsUpdater`) and a network call (`primalPublisher`) while handling potential errors gracefully.

**Example 2: Sending a Zap**

The `zapEvent` method is an even more complex use case, coordinating multiple components to perform a single, seamless action for the user:

```kotlin
override suspend fun zapEvent(
    userId: String,
    walletId: String,
    // ...
): ZapResult {
    // 1. Prepare for an optimistic UI update
    val statsUpdater = target.buildPostStatsUpdaterIfApplicable(userId = userId)
    statsUpdater?.increaseZapStats(...)

    // 2. Create a Nostr Zapper (a wallet interface)
    val nostrZapper = nostrZapperFactory.createOrNull(walletId = walletId)
        ?: return ZapResult.Failure(...)

    // 3. Execute the full zap flow (fetch invoice, pay, etc.)
    val result = nostrZapper.zap(data = ZapRequestData(...))

    // 4. Revert the UI state if the zap fails
    if (result is ZapResult.Failure) {
        statsUpdater?.revertStats()
    }

    return result
}
```

### The Data Layer: Managing the Flow of Information

In Primal's architecture, the `data` module is responsible for one thing: managing data. It orchestrates where data comes from and where it's stored, acting as the concrete implementation of the contracts defined in our `domain` layer. It doesn't care how we display this data, and that shouldn't be a concern when building this part of the app. This module is composed of several key sub-modules:

#### `local` Module: The On-Device Database

This module handles all local data persistence. It uses **Room**, Google's library for managing SQLite databases on Android.

-   **Database Schemas:** Inside the `schemas/` directory, you'll find JSON files that represent each version of our database schema. Room uses these files to handle migrations, ensuring that user data is preserved safely across app updates.
-   **Entities & DAOs:** Here, we define our database tables as `@Entity` classes (e.g., `PostData`, `ProfileData`). For each entity, we have a corresponding **DAO** (Data Access Object), such as `PostDao`. These DAO interfaces provide abstract methods for creating, reading, updating, and deleting data from the database, separating the SQL logic from the rest of the application.

#### `remote` Module: The Network Layer

This is where all communication with external servers happens.

-   **API Definitions:** This module defines interfaces like `FeedApi` and `UsersApi`, which specify the available network calls.
-   **Implementations:** The implementations of these APIs (e.g., `FeedApiImpl`) use the `PrimalApiClient` from the `core` module to execute actual network requests, fetching data from the Primal backend over WebSockets.

#### `repository` Module: The Single Source of Truth

As we've discussed, this is where the **Use Case** logic primarily lives in the Primal project. The repositories are the linchpin of the `data` layer.

-   **Implementing Domain Contracts:** This module contains the implementations of the repository interfaces defined in the `domain` layer (e.g., `FeedRepositoryImpl`).
-   **Orchestrating Data Sources:** Repositories are responsible for coordinating the `local` and `remote` data sources. A typical workflow involves fetching fresh data from the API, storing it in the local Room database for caching, and then providing that data to the rest of the app. This ensures a fast, offline-first user experience.

### The `core` Module: The Foundational Toolkit

It's also important to mention that we have a **`core` layer**. Here, we define the fundamental, low-level components for our mobile application. Think of it as the project's foundation.

This is where we've set up things like `networking-http`, `networking-primal`, and `networking-upload` using the **Ktor** framework. There's also a `utils` package here, which holds all the helper functions we use often throughout the application.

### The Controller Layer: Orchestrating User Actions with ViewModels

In our architecture, the **ViewModel** takes on the role of the "Controller." Its primary job is to manage the flow of user interactions, receive events from the UI, and direct them to the appropriate business logic.

In the Primal application, every screen has a corresponding `ViewModel`. Let's break down its responsibilities:

1.  **Receiving User Input:** When a user clicks a button on the screen (the `View`), that action is sent as an **event** to the `ViewModel`.
2.  **Orchestrating Logic:** The `ViewModel` decides what to do with that event. Using Dependency Injection (DI), it has access to our **repositories** (which contain the use case logic). It calls methods on these repositories to perform specific actions like fetching data, liking a post, or sending a zap.
3.  **Managing State:** The `ViewModel` holds and updates the UI state. After a repository completes its work, the `ViewModel` updates its state, and the Jetpack Compose UI automatically reacts to these changes, ensuring the user always sees the latest information.

By using ViewModels as our controllers, we get the benefit of them being lifecycle-aware, which prevents data loss on configuration changes (like screen rotation) and cleanly separates our UI logic from the business logic.

### The Presentation Layer (UI): Bringing It All to Life

Finally, we arrive at the outermost layer: the **Presentation Layer**. This is our `app` module, and it's the only part of the system the user actually sees and interacts with. Its job is to take the data prepared by the lower layers and present it to the user, as well as to capture user actions and forward them to the `ViewModel`.

This is also where the platform-specific magic happens. Our Android application uses **Jetpack Compose**, while the iOS app uses **SwiftUI**. This means we get a fully native look, feel, and performance on each platform, all powered by the same shared core logic.

### Conclusion: Putting It All Together

By adhering to Clean Architecture, the Primal project achieves a clear separation of concerns.

-   The **`domain`** layer defines the core rules and models, independent of any framework.
-   The **`data`** layer, with its repositories, manages the "how" and "where" of data, acting as the bridge to external sources and containing our use case logic.
-   The **`core`** and **`shared`** modules provide the foundational tools needed for everything to run.
-   Finally, the **`app`** layer, with its **ViewModels (Controllers)** and **Compose UI (Views)**, presents this data to the user and captures their interactions.
