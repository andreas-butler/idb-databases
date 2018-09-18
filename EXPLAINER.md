# Explainer
Documentation & FAQ of observers. See accompanying WebIDL file [IDBObservers.webidl](/IDBObservers.webidl)

**Please file an issue if you have any feedback :)**

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Why?](#why)
- [Example Uses](#example-uses)
  - [Deleting all IndexedDB databases](#deleting-all-indexeddb-databases)
  - [Making interactive list of all IndexedDB databases](#making-interactive-list-of-all-indexeddb-databases)
- [interface IDBObserver](#interface-idbobserver)
  - [new IDBObserver(callback, options)](#new-idbobservercallback-options)
      - [`options` Argument](#options-argument)
  - [IDBObserver.observe(...)](#idbobserverobserve)
    - [`ranges` Argument](#ranges-argument)
  - [IDBObserver.unobserve(database)](#idbobserverunobservedatabase)
  - [Callback Function](#callback-function)
    - [`changes` Argument](#changes-argument)
    - [`records`](#records)
  - [Lifetime](#lifetime)
- [Observation Consistency & Guarantees](#observation-consistency-&-guarantees)
- [Examples (old)](#examples-old)
- [Open Issues](#open-issues)
- [Feature Detection](#feature-detection)
- [Spec changes](#spec-changes)
  - [Observer Creation](#observer-creation)
  - [Change Recording](#change-recording)
  - [Observer Calling](#observer-calling)
- [Future Features](#future-features)
  - [Culling](#culling)
    - [Culling Spec Changes](#culling-spec-changes)
      - [Add Operation](#add-operation)
      - [Put Operation](#put-operation)
      - [Delete Operation](#delete-operation)
      - [Clear Operation](#clear-operation)
- [FAQ](#faq)
    - [Observing onUpgrade](#observing-onupgrade)
    - [Why not expose 'old' values?](#why-not-expose-old-values)
    - [Why not issue 'deletes' instead a 'clear'?](#why-not-issue-deletes-instead-a-clear)
    - [How do I know I have a true state?](#how-do-i-know-i-have-a-true-state)
    - [Why only populate the objectStore name in the `changes` records map?](#why-only-populate-the-objectstore-name-in-the-changes-records-map)
    - [Why not use ES6 Proxies?](#why-not-use-es6-proxies)
    - [What realm are the change objects coming from?](#what-realm-are-the-change-objects-coming-from)
    - [Why not observe from ObjectStore object?](#why-not-observe-from-objectstore-object)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->
# Why?
At the time of writing this explainer IndexedDB does not have a standardized implementation for enumerating all databases within IndexedDB accessible by a given frame. Including such an implementation has been discussed with other browser vendors as well as developers. Both groups have been receptive to implementing such a method.

Use cases for databases enumeration include:
 * Easily deleting all IndexedDB databases for a given frame.
 * Easily opening/closing all IndexedDB databases for a given frame.
 * Operating on programmatically generated IndexedDB databases of variable number.
 * Creating admin UIs that allow for accessing all IndexedDB databases w/o need for going through a tool like DevTools

# Example Uses

## Deleting all IndexedDB databases
There are a number of use cases in which deleting all existing IndexedDB databases is desirable. For example, developers have expressed a desire for wiping all IndexedDB databases in the event of possible data corruption ( https://github.com/w3c/IndexedDB/issues/31#issuecomment-324756055 ). This is implemented simply using IndexedDB database enumeration.
```javascript
let database_info_enumeration = await window.indexedDB.databases();
database_info_enumeration.forEach(function(info) {
  indexedDB.deleteDatabase(info.name);
});
```
## Making interactive list of all IndexedDB databases
One can imagine a case where a developer would like to have access to all indexedDB databases in a simple UI without having to operate through a tool like DevTools (eg: when making an admin page for their website). It could potentially be unreasonable to assume that all the databases available in IndexedDB are explicitly known, and so the only way to implement such a feature would necessitate database enumeration.
```javascript
let database_info_enumeration = await window.indexedDB.databases();
let ui_list = [];
database_info_enumeration.forEach(function(info) {
  ui_list.push(info.name);
});
```

# Database enumeration implementation
The database enumeration function will return a Promise of a Sequence of IDBDatabaseInfo objects. The IDBDatabaseInfo objects will essentially be dictionaries of informational fields relevant to the IndexedDB databases accessible by the frame. The current fields to be implemented are as follows:
  * name: the name of the database
  * version: the indexeddb implementation version of the database
  
Returning the database information as a list wrapped in a Promise was chosen over returning an IDBRequest object, contrary to what has previously been the pattern with IndexedDB functions. This is because web development is moving in the direction of Promise-based asynchronous behaviour and as developers will not have been previously exposed to this standardized database enumeration function it is not expected that breaking the established pattern will cause serious issues. Additionally, keeping the IDBRequest object restricted to interactions with uniquely specified databases (eg: requests for opening / closing particular database instances ) was determined to be a cleaner design choice.

The database version is returned with the name because existing developers desired such a feature ( https://github.com/w3c/IndexedDB/pull/240#issuecomment-391187653 ). Additionally, returning the version with the name forces a more extensible list-like return type that will make adding future fields to the IDBDatabaseInfo Object simpler.


# Observation Consistency & Guarantees
To give the observer strong consistency of the world that it is observing, we need to allow it to
 1. Know the contents of the observing object stores before observation starts (after which all changes will be sent to the observer)
 2. Read the observing object stores at each change observation.

We accomplish #1 by incorporating a transaction into the creation of the observer. After this transaction completes (and has read anything that the observer needs to know), all subsequent changes to the observing object stores will be sent to the observer.

For #2, we optionally allow the observer to
 1. Include the values of the changed keys.  Since we know the initial state, with the keys & values of all changes we can maintain a consistent state of the object stores.
 2. Include a readonly transaction of the observing object stores.  This transaction is scheduled right after the transaction that made these changes, so the object store will be consistent with the 'post observe' world.

# Examples (old)
See the html files for examples, hosted here:
https://dmurph.github.io/indexed-db-observers/

# Open Issues
Issues section here: https://github.com/WICG/indexed-db-observers/issues

# Feature Detection
I'm not sure how we're going to do feature detection. What do you think? What is normal on the web platform?

# Spec changes
These are the approximate spec changes that would happen. See [IDBObservers.webidl](IDBObservers.webidl) for the WebIDL file.

The following extra 'hidden variables' will be kept track of in the spec inside of IDBTransaction:
 * `pending_observer_construction` - a list of {uuid string, options map, callback function} tuples.
 * `change_list` - a per-object-store list of { operation string, optional key IDBKeyRange, optional value object}.  The could be done as a map of object store name to the given list.

## Observer Creation
When the observe method is called on a transaction, the given options, callback, and the transaction's object stores are given a unique id and are stored in a `pending_observer_construction` list as an `Observer`.  This id is given to the observer control, which is returned. When the transaction is completed successfuly, the Observer is added to the domain-specific list of observers.  If the transaction is not completed successfuly, this does not happen.

## Change Recording
Every change would record an entry in the `change_list` for the given object store.

## Observer Calling
When a transaction successfully completes, send the `change_list` changes to all observers that have objects stores touched by the completed transaction. When sending the changes to each observer, all changes to objects stores not observed by the observer are filtered out.


# Future Features

As stated above, future features will potentially involve modifying the IDBDatabaseInfo object so that more meta-information regarding IndexedDB database instances can be returned upon invocation of the database enumeration function.

# FAQ
### Observing onUpgrade
This spec does not offer a way to observe during onupgrade. Potential clients voiced they wouldn't need this feature. This doesn't seem like it's needed either, as one can just read in any data they need with the transaction used to do the upgrading.  Then the observer is guarenteed to begin at the end of that transaction (if one is added), and it wouldn't miss any chanage.

### Why not expose 'old' values?
IndexedDB was designed to allow range delete optimizations so that `delete [0,10000]` doesn't actually have to physically remove those items to return. Instead we can store range delete metadata to shortcut these operations when it makes sense. Since we have many assumptions for this baked our abstraction layer, getting an 'original' or 'old' value would be nontrivial and incur more overhead.

### Why not issue 'deletes' instead a 'clear'?
Following from the answer above, IndexedDB's API is designed to allow mass deletion optimization, and in order to have the 'deletes instead of clear' functionality, this would involve expensive read operations within the database.  If an observer needed to know exactly what was deleted, they can maintain their own state of the keys that they care about.

### How do I know I have a true state?
One might need to guarantee that their observer can see a true, consistent state of the world. This is guarenteed by the way the observer is created within a transaction. When that transaction completes, the observer will be notified for all subsequent transations. Any initial state the observer needs can be read in the initial transaction.

One can also specify the `includeTransaction` option. This means that every observation callback will receive a readonly transaction for the object store/s that it is observing. It can then use this transaction to see the true state of the world. This transaction will take place immediately after the transaction in which the given changes were performed is completed.

### Why only populate the objectStore name in the `changes` records map?
Object store objects are only valid when retrieved from transactions. The only relevant information of that object outside of the transaction is the name of the object store. Since the transaction is optional for the observation callback, we aren't guaranteed to be able to create the IDBObjectStore object for the observer.  However, it is easy for the observer to retrieve this object by
 1. Specifying `includeTransaction` in the options map
 2. calling changes.transaction.objectStore(name)

### Why not use ES6 Proxies?
The two main reasons are:
 1. We need to observe changes across browsing contexts. Passing a proxy across browsing contexts in not possible, and it's infeasible to have every browsing context create a proxy and give it to everyone else (n*n total proxies).
 2. Changes are committed on a per-transaction basis, and can include changes to multiple object stores. We can encompass this is an easy way when we control the observation function, where this would require specialized and complex logic by the client if it was proxy-based.

### What realm are the change objects coming from?
All changes (the keys and values) are structured cloneable, and are cloned from IDB. So they are not coming from a different realm.

### Why not observe from ObjectStore object?
This makes it so developers cannot reliable observe multiple object stores at the same time.  Example:

Given
 * Object stores os1, os2
 * Observers o1, o2 (listening to os1, os2 respectively)

Order of operations
 * T1 modified os1, os2
 * o1 gets changes from T1
 * T2 modifies o1
 * o2 gets changes from T1
 * o1 gets changes from T2

Even if o1 records the changes from T1 for o2, there is no guarantee that o2 it gets the changes from T1 before another transaction changes o1 again.

We also believe that keeping the 'one observer call per transaction commit' keeps observers easy to understand.
