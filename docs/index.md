AutoDispose
===========

![](https://github.com/uber/AutoDispose/workflows/ci.yml/badge.svg?branch=master)

**AutoDispose** is an RxJava 2+ tool for automatically binding the execution of RxJava streams to a 
provided scope via disposal/cancellation.

Overview
--------

Often (especially in mobile applications), Rx subscriptions need to stop in response to some event 
(for instance, when Activity#onStop() executes in an Android app). In order to support this common 
scenario in RxJava 2, we built AutoDispose.

The idea is simple: construct your chain like any other, and then at subscription you simply drop in
the relevant factory call + method for that type as a converter. In everyday use, it 
 usually looks like this: 

```java
myObservable
    .doStuff()
    .as(autoDisposable(this))   // The magic
    .subscribe(s -> ...);
```

By doing this, you will automatically unsubscribe from `myObservable` as indicated by your 
scope - this helps prevent many classes of errors when an observable emits an item, but the actions 
taken in the subscription are no longer valid. For instance, if a network request comes back after a
 UI has already been torn down, the UI can't be updated - this pattern prevents this type of bug.

### `autoDisposable()`

The main entry point is via static factory `autoDisposable()` methods in the `AutoDispose` class. 
There are two overloads: `Completable` and `ScopeProvider`. They return an 
`AutoDisposeConverter` object that implements all the RxJava `Converter` interfaces for use with
the `as()` operator in RxJava types.

#### Completable (as a scope)

The `Completable` semantic is modeled after the `takeUntil()` operator, which accepts an `Observable` 
whose first emission is used as a notification to signal completion. This is logically the 
behavior of a `Single`, so we choose to make that explicit. Since the type doesn't matter, we 
simplify this further to just be a `Completable`, where the scope-end emission is just a completion event.
All scopes in AutoDispose eventually resolve to a `Completable` that emits the end-of-scope notification
in `onComplete`. `onError` will pass through to the underlying subscription.

#### ScopeProvider

```java
public interface ScopeProvider {
  CompletableSource requestScope() throws Exception;
}
```

`ScopeProvider` is an abstraction that allows objects to expose and control and provide their own scopes.
This is particularly useful for objects with simple scopes ("stop when I stop") or very custom state
that requires custom handling.

Note that Exceptions can be thrown in this, and will be routed through `onError()`. If the thrown exception
is an instance of `OutsideScopeException`, it will be routed through any `OutsideScopeHandler`s (more below)
first, and sent through `onError()` if not handled.

### AutoDisposePlugins

Modeled after RxJava's plugins, this allows you to customize the behavior of AutoDispose.

#### OutsideScopeHandler

When a scope is bound to outside of its allowable boundary, `AutoDispose` will send an error event with an
 `OutsideScopeException` to downstream consumers. If you want to customize this behavior, you can use 
 `AutoDisposePlugins#setOutsideScopeHandler` to intercept these exceptions and rethrow something 
 else or nothing at all.

Example
```java
AutoDisposePlugins.setOutsideScopeHandler(t -> {
    // Swallow the exception, or rethrow it, or throw your own!
})
```

A good use case of this is, say, just silently disposing/logging observers outside of scope exceptions in production but crashing on debug.

The supported mechanism to throw this is in `ScopeProvider#requestScope()` implementations.

#### FillInOutsideScopeExceptionStacktraces
 
If you have your own handling of exceptions in scope boundary events, you can optionally set
`AutoDisposePlugins#setFillInOutsideScopeExceptionStacktraces` to `false`. This will result in 
AutoDispose `not` filling in stacktraces for exceptions, for a potential minor performance boost.

### AutoDisposeAndroidPlugins

Similar to `AutoDisposePlugins`, this allows you to customize the behavior of AutoDispose in Android environments.

#### MainThreadChecker

This plugin allows for supplying a custom `BooleanSupplier` that can customize how main thread 
checks work. The conventional use case of this is Android JUnit tests, where the `Looper` class is 
not stubbed in the mock android.jar and fails explosively when touched.

Another potential use of this at runtime to customize checks for more fine-grained main thread 
checks behavior.

Example
```java
AutoDisposeAndroidPlugins.setOnCheckMainThread(() -> {
    return true; // Use whatever heuristics you prefer.
})
```

### Behavior

Under the hood, AutoDispose decorates RxJava's real observer with a custom *AutoDisposing* observer.
This custom observer leverages the scope to create a disposable, auto-disposing observer that acts 
as a lambda observer (pass-through) unless the underlying scope `CompletableSource` emits `onComplete`. Both 
scope emission and upstream termination result in immediate disposable of both the underlying scope
subscription and upstream disposable.

These custom `AutoDisposing` observers are considered public read-only API, and can be found under the 
`observers` package. They also support retrieval of the underlying observer via `delegateObserver()`
methods. Read-only API means that the public signatures will follow semantic versioning, but we may
add new methods in the future (which would break compilation if you make custom implementations!).

To read this information, you can use RxJava's `onSubscribe` hooks in `RxJavaPlugins` to watch for
instances of these observers.

### Support/Extensions

`Flowable`, `ParallelFlowable`, `Observable`, `Maybe`, `Single`, and `Completable` are all supported. Implementation is solely
based on their `Observer` types, so conceivably any type that uses those for subscription should work.

####  Extensions

There are also a number of extension artifacts available, detailed below.

##### LifecycleScopeProvider

```java
public interface LifecycleScopeProvider<E> extends ScopeProvider {
  Observable<E> lifecycle();

  Function<E, E> correspondingEvents();

  E peekLifecycle();
  
  // Inherited from ScopeProvider
  CompletableSource requestScope();
}
```

A common use case for this is objects that have implicit lifecycles, such as Android's `Activity`, 
`Fragment`, and `View` classes. Internally at subscription-time, `AutoDispose` will resolve
a `CompletableSource` representation of the target `end` event in the lifecycle, and exposes an API to dictate what
corresponding events are for the current lifecycle state (e.g. `ATTACH` -> `DETACH`). This also allows
you to enforce lifecycle boundary requirements, and by default will error if the lifecycle has either
not started yet or has already ended.

`LifecycleScopeProvider` is a special case targeted at binding to things with lifecycles. Its API is
as follows:
  - `lifecycle()` - returns an `Observable` of lifecycle events. This should be backed by a `BehaviorSubject`
  or something similar (`BehaviorRelay`, etc).
  - `correspondingEvents()` - a mapping of events to corresponding ones, i.e. Attach -> Detach.
  - `peekLifecycle()` - returns the current lifecycle state of the object.

In `requestScope()`, the implementation expects to these pieces to construct a `CompletableSource` representation 
of the proper end scope, while also doing precondition checks for lifecycle boundaries. If a 
lifecycle has not started, it will send you to `onError` with a `LifecycleNotStartedException`. If 
the lifecycle as ended, it is recommended to throw a `LifecycleEndedException` in your 
`correspondingEvents()` mapping, but it is up to the user.

To simplify implementations, there's an included `LifecycleScopes` utility class with factories 
for generating `CompletableSource` representations from `LifecycleScopeProvider` instances.

`autodispose-lifecycle` contains the core `LifecycleScopeProvider` and `LifecycleScopes` APIs as well as a convenience test helper.

##### Android

There are three artifacts with extra support for Android:
* `autodispose-android` has a `ViewScopeProvider` for use with Android `View` classes.
* `autodispose-androidx-lifecycle` has a `AndroidLifecycleScopeProvider` for use with `LifecycleOwner` and `Lifecycle` implementations.

Note that the project is compiled against Java 8. If you need support for lower Java versions, you should
use D8 (Android Gradle Plugin 3.2+) or desugar as needed (depending on the build system).

##### Kotlin

Kotlin extensions are bundled with almost every artifact.

For coroutines - there is an `autodispose-coroutines-interop` artifact for interoperability between
`CoroutineScope` and `ScopeProvider`/`Completable` types.

##### RxLifecycle

As of 0.4.0 there is an RxLifecycle interop module under `autodispose-rxlifecycle`. This is for interop
with [RxLifecycle](https://github.com/trello/RxLifecycle)'s `LifecycleProvider` interfaces.

### Philosophy

Each factory returns a subscribe proxies upon application that just proxy to real subscribe calls under 
the hood to "AutoDisposing" implementations of the types. These types decorate the actual observer 
at subscribe-time to achieve autodispose behavior. The types are *not* exposed directly because
autodisposing has *ordering* requirements; specifically, it has to be done at the end of a chain to properly
wrap all the upstream behavior. Lint could catch this too, but we have seen no use cases for disposing 
upstream (which can cause a lot of unexpected behavior). Thus, we optimize for the common case, and the
API is designed to prevent ordering issues while still being a drop-in one-liner.

## Motivations

Lifecycle management with RxJava and Android is nothing new, so why yet another tool?

Two common patterns for binding execution in RxJava that we used prior to this were as follows:

* `CompositeSubscription` field that all subscriptions had to be manually added to.
* `RxLifecycle`, which works via `compose()` to resolve the lifecycle end event and ultimately transform the
given observable to `takeUntil()` that event is emitted.

Both implementations are elegant and work well, but came with caveats that we sought to revisit and solve
in AutoDispose. 

`CompositeSubscription` requires manual capture of the return value of `subscribe` calls, and
gets tedious to reason about with regards to binding subscription until different events.

[`RxLifecycle`][rxlifecycle] solves the caveats of `CompositeSubscription` use by working in a dead-simple API and handling
resolution of corresponding events. It works great for `Observable` types, but due to the nature of 
how `takeUntil()` works, we found that `Single` and `Completable` usage was risky to use (particularly in a 
 large team with varying levels of RxJava experience) considering lifecycle interruption would result
in a downstream `CancellationException` every time. It's the contract of those types, but induced a lot of
ceremony for what would otherwise likely be our most commonly used type (`Single`). Even with `Observable`,
we were still burned occasionally by the completion event still coming through to an unsuspecting engineer.
Another caveat we often ran into (and later aggressively linted against) was that the `compose()` call had
ordering implications, and needed to be as close to the `subscribe()` call as possible to properly wrap upstream.
If binding to views, there were also threading requirements on the observable chain in order to work properly.
 
At the end of the day, we wanted true disposal/unsubscription-based behavior, but with RxLifecycle-esque
semantics around scope resolution. RxJava 2's `Observer` interfaces provide the perfect mechanism for
 this via their `onSubscribe()` callbacks. The result is de-risked `Single`/`Completable` usage, no ordering
 concerns, no threading concerns (fingers crossed), and true disposal with no further events of any kind
 upon scope end. We're quite happy with it, and hope the community finds it useful as well.
 
Special thanks go to [Dan Lew][dan] (creator of RxLifecycle), who helped pioneer this area for RxJava 
 in android and humored many of the discussions around lifecycle handling over the past couple years 
 that we've learned from. Much of the internal scope resolution mechanics of `AutoDispose` are 
 inspired by RxLifecycle.
 
## RxJava versions support

- RxJava 3
  - AutoDispose 2.x
- RxJava 2
  - AutoDispose 1.x
- RxJava 1
  - We do not plan to try to backport this to RxJava 1. This pattern is *sort of* possible in 
  RxJava 1, but only on `Subscriber` (via `onStart()`) and `CompletableObserver` (which matches the 
  API of RxJava 2+).

2.x versions of AutoDispose are built for RxJava 3.

## Static analysis

### Error Prone

There is an optional error-prone checker you can use to enforce use of AutoDispose.
Integration steps and more details
can be found on the [website](https://uber.github.io/AutoDispose/error-prone)

### Lint Check

AutoDispose ships with a lint check that detects missing AutoDispose scope within defined scoped elements. Integration steps and more details can be found on the [website](https://uber.github.io/AutoDispose/lint-check/)

Download
--------

Java:

[![Maven Central](https://img.shields.io/maven-central/v/com.uber.autodispose2/autodispose.svg)](https://mvnrepository.com/artifact/com.uber.autodispose2/autodispose)

```gradle
implementation 'com.uber.autodispose2:autodispose:x.y.z'
```

LifecycleScopeProvider:

[![Maven Central](https://img.shields.io/maven-central/v/com.uber.autodispose2/autodispose-lifecycle.svg)](https://mvnrepository.com/artifact/com.uber.autodispose2/autodispose-lifecycle)
```gradle
implementation 'com.uber.autodispose2:autodispose-lifecycle:x.y.z'
```

Android extensions:

[![Maven Central](https://img.shields.io/maven-central/v/com.uber.autodispose2/autodispose-android.svg)](https://mvnrepository.com/artifact/com.uber.autodispose2/autodispose-android)
```gradle
implementation 'com.uber.autodispose2:autodispose-android:x.y.z'
```

Android Architecture Components extensions:

[![Maven Central](https://img.shields.io/maven-central/v/com.uber.autodispose2/autodispose-androidx-lifecycle.svg)](https://mvnrepository.com/artifact/com.uber.autodispose2/autodispose-android-archcomponents)
```gradle
// AutoDispose 1.x
implementation 'com.uber.autodispose:autodispose-android-archcomponents:x.y.z'

// AutoDispose 2.x
implementation 'com.uber.autodispose2:autodispose-androidx-lifecycle:x.y.z'
```

Androidx-Lifecycle Test extensions:

[![Maven Central](https://img.shields.io/maven-central/v/com.uber.autodispose2/autodispose-androidx-lifecycle-test.svg)](https://mvnrepository.com/artifact/com.uber.autodispose2/autodispose-androidx-lifecycle-test)
```gradle
// AutoDispose 1.x
implementation 'com.uber.autodispose:autodispose-android-archcomponents-test:x.y.z'

// AutoDispose 2.x
implementation 'com.uber.autodispose2:autodispose-androidx-lifecycle-test:x.y.z'
```

RxLifecycle interop (AutoDispose 1.x/RxJava 2.x only):

`autodispose-rxlifecycle` [![Maven Central](https://img.shields.io/maven-central/v/com.uber.autodispose/autodispose-rxlifecycle.svg)](https://mvnrepository.com/artifact/com.uber.autodispose/autodispose-rxlifecycle)
```gradle
implementation 'com.uber.autodispose:autodispose-rxlifecycle:x.y.z'
```

`autodispose-rxlifecycle3` [![Maven Central](https://img.shields.io/maven-central/v/com.uber.autodispose/autodispose-rxlifecycle3.svg)](https://mvnrepository.com/artifact/com.uber.autodispose/autodispose-rxlifecycle3)
```gradle
implementation 'com.uber.autodispose:autodispose-rxlifecycle3:x.y.z'
```

Javadocs and KDocs for the most recent release can be found here: https://uber.github.io/AutoDispose/2.x/autodispose/

Snapshots of the development version are available in [Sonatype's snapshots repository][snapshots].

 [rxlifecycle]: https://github.com/trello/RxLifecycle/
 [dan]: https://twitter.com/danlew42
 [snapshots]: https://oss.sonatype.org/content/repositories/snapshots/com/uber/autodispose2/
