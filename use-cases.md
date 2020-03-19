# Introduction

The WeakRefs proposal adds WeakRefs and FinalizationRegistries to the standard library.
This page provides use cases and patterns to avoid for them.

* [Use Cases for WeakRefs](#use-cases-for-weakrefs)
    * [Caches of Large or Expensive-to-Retrieve Objects](#caches-of-large-or-expensive-to-retrieve-objects)
* [Patterns to Avoid With WeakRefs](#patterns-to-avoid-with-weakrefs)
    * [Scanning for WeakRefs Whose Targets Have Been Reclaimed](#scanning-for-weakrefs-whose-targets-have-been-reclaimed)
* [Use Cases for Cleanup Callbacks](#use-cases-for-cleanup-callbacks)
    * [Cleanup in Caches of Large or Expensive-to-Retrieve Objects](#cleanup-in-caches-of-large-or-expensive-to-retrieve-objects)
    * [Exposing Web Assembly Memory to JavaScript](#exposing-web-assembly-memory-to-javascript)
    * [Cleaning Up Cross-Process Resources](#cleaning-up-cross-process-resources)
* [Patterns to Avoid With Cleanup Callbacks](#patterns-to-avoid-with-cleanup-callbacks)
    * [Using Cleanup Callbacks to Release External Resources](#using-cleanup-callbacks-to-release-external-resources)



----

# Warning - Avoid Where Possible

Please read [the explainer](./README.md) for several words of caution regarding weak references and cleanup callbacks. Their correct use takes careful thought, and they are best avoided if possible. It's also important to avoid relying on any specific behaviors not guaranteed by the specification. When, how, and whether garbage collection occurs is down to the implementation of any given JavaScript engine. Any behavior you observe in one engine may be different in another engine, in another version of the same engine, or even in a slightly different situation with the same version of the same engine. Garbage collection is a hard problem that JavaScript engine implementers are constantly refining and improving their solutions to.

----

## Use Cases for WeakRefs

Here are some use cases for WeakRefs.

### Caches of Large or Expensive-to-Retrieve Objects

**TBD**

## Patterns to Avoid With WeakRefs

### Scanning for WeakRefs Whose Targets Have Been Reclaimed

Since *cleanup callbacks* (see [`FinalizationRegistry`](#finalizationregistry)) may never be called, it would be understandable for a developer to think:

> Ah! So if I have a bunch of WeakRef instances, I should have some kind of proactive cleanup I perform, scanning through them looking for `deref` returning `undefined`, so that when their targets have been reclaimed, I release those WeakRef instances and any other bookkeeping information I have for them to get that memory back.

**That isn't recommended.** It adds complication and runtime cost to the code. It's likely that an engine that performs garbage collection will, eventually, call your cleanup callbacks, or that you don't need it to (process shutdown, etc.). It's not guaranteed, but the cost of some wasted WeakRef instances is generally not worth the complexity of performing this kind of scan.

## Use Cases for Cleanup Callbacks

Here are some use cases for cleanup callbacks.

### Cleanup in Caches of Large or Expensive-to-Retrieve Objects

**TBD**: Point to the WeakRef use case and explain about cleaning up.

### Exposing Web Assembly Memory to JavaScript

**TBD**: Mostly copying the README.md.

### Cleaning Up Cross-Process Resources

**TBD**: Mostly copying the README.md about proxies for worker objects, but slightly more generalized with that being an example.

## Patterns to Avoid With Cleanup Callbacks

Here are some things to avoid with cleanup callbacks.

### Using Cleanup Callbacks to Release External Resources

Don't rely on cleanup callbacks to release external resources. For example, consider the `FileStream` API in [the examples](examples.md#locating-and-responding-to-resource-leaks). The API has an explicit `close` method that developers are expected to call in the normal course of using the API. It does use cleanup callbacks, but it does so primarily to aid developers in using the API correctly by notifying them that they've failed to call `close`. While it could *also* release the file handles or other OS resources via the cleanup callback, that's not the cleanup callback's primary purpose. Its primary purpose is to help developers use the API correctly.

<!-- for scrolling to links near the end -->
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>