# Explainer
Documentation & FAQ of IndexedDB databases enumeration function. 
**Please file an issue @ [https://github.com/w3c/IndexedDB/issues](#https://github.com/w3c/IndexedDB/issues) if you have any feedback :)**

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Why?](#why)
- [Example Uses](#example-uses)
  - [Deleting all IndexedDB databases](#deleting-all-indexeddb-databases)
  - [Making interactive list of all IndexedDB databases](#making-interactive-list-of-all-indexeddb-databases)
- [Database enumeration implementation](#database-enumeration-implementation)
- [Database enumeration consistency and guarantees](#database-enumeration-consistency-and-guarantees)
- [Spec changes](#spec-changes)
- [Future features](#future-features)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->
# Why?
IndexedDB databases() provides a standardized method for enumerating all IndexedDB databases accessible by the current context, allowing for easy iteration over databases without requiring explicit knowledge of database names. At the time of writing this explainer such a standardized method does not exist. Including such an implementation has been discussed amongst browser vendors as well as developers. Both groups have been receptive to implementing such a method  ( [https://github.com/w3c/IndexedDB/issues/31](#https://github.com/w3c/IndexedDB/issues/31) ).

Use cases for databases enumeration include:
 * Easily deleting all IndexedDB databases accessible by a given context.
 * Easily opening/closing all IndexedDB databases accessible by a given context.
 * Operating on programmatically generated IndexedDB databases of variable number.
 * Creating admin UIs that allow for accessing all IndexedDB databases w/o need for going through a tool like DevTools

# Example Uses

## Deleting all IndexedDB databases
There are a number of use cases in which deleting all existing IndexedDB databases is desirable. For example, developers have expressed a desire for wiping all IndexedDB databases in the event of possible data corruption ( [https://github.com/w3c/IndexedDB/issues/31#issuecomment-324756055](#https://github.com/w3c/IndexedDB/issues/31#issuecomment-324756055) ). This is implemented simply using IndexedDB database enumeration.
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

The database version is returned with the name because existing developers desired such a feature ( [https://github.com/w3c/IndexedDB/pull/240#issuecomment-391187653](#https://github.com/w3c/IndexedDB/pull/240#issuecomment-391187653) ). Additionally, returning the version with the name forces a more extensible list-like return type that will make adding future fields to the IDBDatabaseInfo Object simpler.

# Database enumeration consistency and guarantees
The data returned by the database enumeration function is a snapshot. The database information is collected asynchronously and so there are no guarantees regarding the sequencing of collection with respect to any additional requests to create/delete/upgrade IndexedDB databases.

This asynchronous nature of the enumeration function resulted in a delay in its eventual implementation in a standardized form due to objections to racy behaviour.

# Spec changes
See [https://github.com/w3c/IndexedDB/pull/240/files#diff-ec9cfa5f3f35ec1f84feb2e59686c34dR2369](#https://github.com/w3c/IndexedDB/pull/240/files#diff-ec9cfa5f3f35ec1f84feb2e59686c34dR2369)

# Future Features
As stated above, future features will potentially involve modifying the IDBDatabaseInfo object so that more meta-information regarding IndexedDB database instances can be returned upon invocation of the database enumeration function.
