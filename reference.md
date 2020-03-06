# Introduction

The WeakRefs proposal adds WeakRefs and FinalizationRegistries to the standard library.

* [WeakRef](#weakref) - weak references
* [FinalizationRegistry](#finalizationregistry) - registries providing cleanup callbacks

This page provides developer reference information, use cases, and best practices for them.

----

# Warning - Avoid Where Possible

Please read [the explainer](./README.md) for several words of caution regarding weak references and cleanup callbacks. Their correct use takes careful thought, and they are best avoided if possible. It's also important to avoid relying on any specific behaviors not guaranteed by the specification. When, how, and whether garbage collection occurs is down to the implementation of any given JavaScript engine. Any behavior you observe in one engine may be different in another engine, in another version of the same engine, or even in a slightly different situation with the same version of the same engine. Garbage collection is a hard problem that JavaScript engine implementers are constantly refining and improving their solutions to.

----

# `WeakRef`

A WeakRef instance contains a weak reference to an object, which is called its *target* or *referent*. A *weak reference* to an object is a reference to it that does not prevent it being reclaimed by the garbage collector. In contrast, a normal (or *strong*) reference keeps an object in memory. When an object no longer has any strong references to it, the JavaScript engine's garbage collector may destroy the object and reclaim its memory. If that happens, you can't get the object from a weak reference anymore.

**WeakRef Contents:**

* [Overview](#weakref-overview)
* [State Guarantee](#state-guarantee)
* [Other Guarantees](#other-guarantees)
* [WeakRef API](#weakref-api)
    * [The `WeakRef` Constructor](#the-weakref-constructor)
    * [`WeakRef.prototype.deref`](#weakrefprototypederef)
* [Use Cases for WeakRefs](#use-cases-for-weakrefs)
    * [Caches of Large or Expensive-to-Retrieve Objects](#caches-of-large-or-expensive-to-retrieve-objects)
* [Patterns to Avoid With WeakRefs](#patterns-to-avoid-with-weakrefs)

## WeakRef Overview

To create a WeakRef, you use the `WeakRef` constructor, passing in the target object (the object you want a weak reference to):

```js
let ref = new WeakRef(someObject);
```

You generally release your strong reference (`someObject`) at some point while keeping the WeakRef instance (`ref`).

To get the object from a WeakRef instance, use its `deref` method:

```js
let obj = ref.deref();
if (obj) {
    // ...use `obj`...
}
```

`deref` returns the object, or `undefined` if the object is no longer available.

## State Guarantee

A WeakRef instance's state will not observably change during an active JavaScript [job](https://tc39.es/ecma262/#job) (including any promise reaction jobs that run at the end of a script job). That means if you create a WeakRef instance and call `deref` within the same job, you are guaranteed to receive a reference to the object. Also, if you call `deref` on a WeakRef and get an object, a second call to `deref` on that same WeakRef during the same job will also return that same object (not `undefined`). The state of a WeakRef can only be observed to change between one job and another (between "turns" of the event loop).

For example, this is guaranteed to work (though is arguably not best practice):

```js
let ref = new WeakRef({example: "value"});
console.log(ref.deref().example); // "value"
```

The target object will not be reclaimed between the creation of the WeakRef and the call to `deref`.

Similarly, while again arguably not best practice, this is guaranteed to work:

```js
if (ref.deref()) {
    const { someProp } = ref.deref();
    // ...
}
```

The target object will not be reclaimed after the first call to `deref` and before the second call to `deref`, so the second call above will never return `undefined`.

## Other Guarantees

Conforming JavaScript engines are not required to do garbage collection at all (for instance, some implementations for embedded systems may not). But if an engine does garbage collect the target of a WeakRef, the next call to `deref` on that WeakRef is guaranteed to return `undefined` as opposed to some kind of error or returning an invalid object.

## WeakRef API

### The `WeakRef` Constructor

The `WeakRef` constructor creates a WeakRef instance for the given `target` object:

```js
ref = new WeakRef(target)
```

**Parameters**

* `target` - The target object for the WeakRef instance.

**Returns**

* The WeakRef instance.

**Errors**

* `TypeError` if `target` isn't an object.

**Notes**

* `WeakRef` throws if called as a function rather than as a constructor.
* `WeakRef` is designed to be subclassed, and so can appear in the `extends` of a `class` definition and similar.

### `WeakRef.prototype.deref`

The `deref` method returns the WeakRef instance's target object, or `undefined` if the target object has been reclaimed:

```js
targetOrUndefined = weakRef.deref();
```

See the [*State Guarantee*](#state-guarantee) section for guarantees around the stability of its return value.

**Parameters**

* *(none)*

**Returns**

* The target object of the WeakRef, or `undefined`.

**Errors**

* *(none)*

## Use Cases for WeakRefs

Here are some use cases for WeakRefs.

### Caches of Large or Expensive-to-Retrieve Objects

**TBD**

## Patterns to Avoid With WeakRefs

### Scanning for WeakRefs Whose Targets Have Been Reclaimed

Since *cleanup callbacks* (see [`FinalizationRegistry`](#finalizationregistry)) may never be called, it would be understandable for a developer to think:

> Ah! So if I have a bunch of WeakRef instances, I should have some kind of proactive cleanup I perform, scanning through them looking for `deref` returning `undefined`, so that when their targets have been reclaimed, I release those WeakRef instances and any other bookkeeping information I have for them to get that memory back.

**That isn't recommended.** It adds complication and runtime cost to the code. It's likely that an engine that performs garbage collection will, eventually, call your cleanup callbacks, or that you don't need it to (process shutdown, etc.). It's not guaranteed, but the cost of some wasted WeakRef instances is generally not worth the complexity of performing this kind of scan.

# `FinalizationRegistry`

A finalization registry provides a means of requesting that a *cleanup callback* get called at some point after an object registered with the registry has been *reclaimed* (garbage collected). (Cleanup callbacks are sometimes called *finalizers*.)

**NOTE:** The form of this API is under review, see [issue #187](https://github.com/tc39/proposal-weakrefs/pull/187).

**NOTE:** Cleanup callbacks should not be used for essential program logic. The [*Cleanup Callback Guarantees*](#cleanup-callback-guarantees) section below goes into more detail about why.

**FinalizationRegistry Contents**:

* [Overview](#finalizationregistry-overview)
* [Cleanup Callback Guarantees](#cleanup-callback-guarantees)
* [FinalizationRegistry API](#finalizationregistry-api)
    * [The `FinalizationRegistry` Constructor](#the-finalizationregistry-constructor)
    * [`FinalizationRegistry.prototype.register`](#finalizationregistryprototyperegister)
    * [`FinalizationRegistry.prototype.unregister`](#finalizationregistryprototypeunregister)
    * [`FinalizationRegistry.prototype.cleanupSome`](#finalizationregistryprototypecleanupsome)
* [Use Cases for Cleanup Callbacks](#use-cases-for-cleanup-callbacks)
    * [Cleanup in Caches of Large or Expensive-to-Retrieve Objects](#cleanup-in-caches-of-large-or-expensive-to-retrieve-objects)
    * [Exposing Web Assembly Memory to JavaScript](#exposing-web-assembly-memory-to-javascript)
    * [Cleaning Up Cross-Process Resources](#cleaning-up-cross-process-resources)
* [Patterns to Avoid With Cleanup Callbacks](#patterns-to-avoid-with-cleanup-callbacks)
    * [Using Cleanup Callbacks to Release External Resources](#using-cleanup-callbacks-to-release-external-resources)

## FinalizationRegistry Overview

A `FinalizationRegistry` instance (a "registry") lets you get *cleanup callbacks* after objects registered with the registry are reclaimed. You create the registry passing in the callback:

```js
const registry = new FInalizationRegistry(heldValue => {
    // ....
});
```

Then you register any objects you want a cleanup callback for by calling the `register` method, passing in the object and a *held value* for it:

```js
registry.register(theObject, "some value");
```

The registry does not keep a strong reference to the object, as that would defeat the purpose (if the registry held it strongly, the object would never be reclaimed).

If `theObject` is reclaimed, your cleanup callback may be called at some point in the future with the *held value* you provided for it (`"some value"` in the above). The held value can be any value you like, a primitive or an object, even `undefined`. If the held value is an object, the registry keeps a *strong* reference to it (so it can pass it to your cleanup callback later).

If you might want to unregister an object later, you pass a third value, which is the *unregistration token* you'll use later when calling the registry's `unregister` function to unregister the object.

It's common to use the object itself as the unregister token, which is just fine:

```js
registry.register(theObject, "some value", theObject);
// ...some time later, if you don't care about `theObject` anymore...
registry.unregister(theObject);
```

But it doesn't have to be the same object, it can be a different one:

```js
registry.register(theObject, "some value", tokenObject);
// ...some time later, if you don't care about `theObject` anymore...
registry.unregister(tokenObject);
```

The registry keeps only a weak reference to the unregistration token.

## Cleanup Callback Guarantees

Developers shouldn't rely on cleanup callbacks for essential program logic. Cleanup callbacks may be useful for reducing memory usage across the course of a program, but are unlikely to be useful otherwise.

A conforming JavaScript implementation, even one that does garbage collection, is not required to call cleanup callbacks. When and whether it does so is entirely down to the implementation of the JavaScript engine. When a registered object is reclaimed, the cleanup callbacks associated with any registries it's registered with may be called some time later, or not at all.

It's likely that major implementations will call cleanup callbacks at some point during execution, but those calls may be substantially after the related object was reclaimed.

There are also situations where even implementations that normally call cleanup callbacks are unlikely to call them:

* When the JavaScript realm the objects existed in is being destroyed entirely (for instance, closing a tab in a browser).
* When the `FinalizationRegistry` instance itself is no longer reachable by JavaScript code.

See [Use Cases for Cleanup Callbacks](#use-cases-for-cleanup-callbacks) and [Patterns to Avoid With Cleanup Callbacks](#patterns-to-avoid-with-cleanup-callbacks) for more on this topic.

## FinalizationRegistry API

### The `FinalizationRegistry` Constructor

Creates a finalization registry with an associated cleanup callback:

```js
registry = new FinalizationRegistry(cleanupCallback)
```

**Parameters**

* `cleanupCallback` - The callback to call after an object in the registry has been reclaimed. This is required and must be callable.

**Returns**

* The FinalizationRegistry instance.

**Errors**

* `TypeError` if `cleanupCallback` isn't callable.

**Notes**

* `FinalizationRegistry` is designed to be subclassed, and so can appear in the `extends` of a `class` definition and similar.

### `FinalizationRegistry.prototype.register`

Registers an object with the registry:

```js
registry.register(target, heldValue[, unregisterToken])
```

**Parameters**

* `target` - The target object to register. (Held weakly by the registry.)
* `heldValue` - The value to pass to the finalizer for this object. (Held strongly by the registry.) This cannot be the `target` object.
* `unregisterToken` - (Optional) The token to pass to the `unregister` method to unregister the target object. If provided, this must be an object. If not provided, the target cannot be unregistered. (Held weakly by the registry.)

**Returns**

* *(none)*

**Errors**

* `TypeError` if any of the following is true:
    * `target` isn't an object.
    * `target` and `heldValue` are the same object.
    * `unregisterToken` is not `undefined` and not an object.

### `FinalizationRegistry.prototype.unregister`

Unregisters an object from the registry:

```js
registry.unregister(unregisterToken)
```

**Parameters**

* `unregisterToken` - The token that was used as the `unregisterToken` argument when calling `regiter` to register the target object.

**Returns**

* *(none)*

**Errors**

* `TypeError` if `unregisterToken` isn't an object.

**Notes**

When a target object has been reclaimed and its cleanup callback has been called, it is no longer registered in the registry. There is no need to all `unregister` in your cleanup callback. Only call `unregister` if you haven't received a cleanup callback and no longer need to receive one.

### `FinalizationRegistry.prototype.cleanupSome`

Triggers callbacks for an implementation-chosen number of objects in the registry that have been reclaimed but whose callbacks have not yet been called:

```js
registry.cleanupSome([callback])
```

**NOTE:** Normally, you don't call this function. Leave it to the JavaScript engine's garbage collector to do the cleanup as appropriate. This function primarily exists to support long-running jobs in Wasm applications. Use in JavaScript code is likely to be unnecessary.

**Parameters**

* `callback` - (Optional) If given, the registry uses the given callback instead of the one it was created with.

**Returns**

* *(none)*

**Errors**

* `TypeError` if `callback` isn't `undefined` and isn't callable.

**Notes**

* The number of entries for reclaimed objects that are cleaned up from the registry (calling the cleanup callbacks) is implementation-defined. An implementation might remove just one eligible entry, or all eligible entries, or somewhere in between.

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