# ES6 Modules

Status | Draft
------ | ----
Author | Bradley Meck
Date   | 7 Jan 2016

It is our intent to:

* implement interoperability for ES6 modules and node's existing module system
* create a **Registry Object** (see WhatWG section below) compatible with the WhatWG Loader Registry

## Purpose

1. Allow a common module syntax for Browser and Server.
2. Allow a common registry for inspection by Browser and Server environments/tools.
    * these will most likely be represented by metaproperties like `import.context` but spec is not in place fully yet.

## Related

---

### [ECMA262](https://tc39.github.io/ecma262/#sec-source-text-module-records)

---

Discusses the syntax and semantics of related syntax, and introduces:
    
#### Types
    
* [ModuleRecord](https://tc39.github.io/ecma262/#sec-abstract-module-records)
    - Defines the list of imports via [ImportEntry](https://tc39.github.io/ecma262/#table-39).
    - Defines the list of exports via [ExportEntry](https://tc39.github.io/ecma262/#table-41).
    
* [ModuleNamespace](https://tc39.github.io/ecma262/#sec-module-namespace-exotic-objects)
    - Represents a read-only snapshot of a module's exports.
    
### Operations
    
* [ParseModule](https://tc39.github.io/ecma262/#sec-parsemodule)
    - Creates a SourceTextModuleRecord from source code.
    
* [HostResolveImportedModule](https://tc39.github.io/ecma262/#sec-hostresolveimportedmodule)
    - A hook for when an import is exactly performed.
    
* [CreateImportBinding](https://tc39.github.io/ecma262/#sec-createimportbinding)
    - A means to create a shared binding (variable) from an export to an import. Required for the "live" binding of imports.

* [ModuleNamespaceCreate](https://tc39.github.io/ecma262/#sec-modulenamespacecreate)
    - Provides a means of creating a list of exports manually, used so that CommonJS `module.exports` can create ModuleRecords that are prepopulated.

---

### [WhatWG Loader](https://github.com/whatwg/loader)

---

Discusses the design of module metadata in a [Registry](https://whatwg.github.io/loader/#registry). All actions regarding the Registry are synchronous.

**NOTE** It is not Node's intent to implement the asynchronous pipeline in the Loader specification. There is discussion about including a synchronous pipeline in the specification as an addendum.


### [Summary Video on Youtube](youtube.com/watch?v=NdOKO-6Ty7k)

## Additional Structures Required

### DynamicModuleRecord

A Module Record that presents a view of an Object for its `[[Namespace]]` rather than coming from an environment record.

The list of exports is frozen upon construction. No new properties may be added. No properties may be removed.

## Algorithm

When `require()`ing a file.

1. Determine if file is ES6 or CommonJS (CJS).
2. If CJS
	1. Evaluate immediately
	2. Produce a DynamicModuleRecord from `module.exports`
3. If ES6
	1. Parse for `import`/`export`s and keep record, in order to create bindings
	2. Gather all submodules by performing `require` recursively
		* See circular dep semantics below 
	3. Connect `import` bindings for all relevant submodules (see [ModuleDeclarationInstantiation](https://tc39.github.io/ecma262/#sec-moduledeclarationinstantiation))
	4. Evaluate

## Semantics

### Determining if source is an ES6 Module

If a module can be parsed outside of the ES6 Module goal, it will be treated as a [Script](https://tc39.github.io/ecma262/#sec-parse-script). Otherwise it will be parsed as [ES6](https://tc39.github.io/ecma262/#sec-parsemodule).

In pseudocode, it looks somewhat like:

```
try {
  v8::Script::Compile
}
catch (e) {
  v8::Script::CompileModule
}
```

V8 may or may not choose to use parser fallback to combine this into one step.

### CommonJS consuming ES6

#### default exports

ES6 modules only ever declare named exports. A default export just exports a property named `default`.

Given

```javascript
let foo = 'my-default';
default export foo;
```

```javascript
require('./es6');
// {default:'my-default'}
```

#### read only

The objects create by an ES6 module are [ModuleNamespace Objects](https://tc39.github.io/ecma262/#sec-module-namespace-exotic-objects).

These have `[[Set]]` be a no-op and are read only views of the exports of an ES6 module.

### ES6 consuming CommonJS

#### default imports

`module.exports` shadows `module.exports.default`.

This means that if there is a `.default` on your CommonJS exports it will be shadowed to the `module.exports` of your module.

Given:

```javascript
// cjs.js
module.exports = {default:'my-default'};
```

You will grab `module.exports` when performing an ES6 import.

```javascript
// es6.js
import foo from './cjs';
// foo = {default:'my-default'};

import {default as bar} from './cjs';
// bar = {default:'my-default'};
```

### Known Gotchas

All of these gotchas relate to opt-in semantics and the fact that CommonJS is a dynamic loader while ES6 is a static loader.

No existing code will be affected.

#### Circular Dep CJS => ES6 => CJS

Given:

```javascript
//cjs.js
module.exports = {x:0};
require('./es6');
```

```javascript
//es6.js
import * as ns from './cjs';
// ns = undefined
import cjs from './cjs';
// ns = undefined
```

The list of properties being exported is not populated until `cjs.js` finishes executing.

The result is that there are no properties to import and you recieve an empty module.

##### Option: Throw on this case

Since this case is coming from ES6, we could throw whenever this occurs. This is similar to how you cannot change a generator while it is running.

If taken this would change the ES6 module behavior to:

```javascript
//es6.js
import * as ns from './cjs';
// throw new EvalError('./cjs is not an ES6 module and has not finished evaluation');
```

## Advisory

V8 currently does not expose the proper APIs for creating Loaders, it has done the brunt of the work by [exposing a parser in their issue tracker](https://bugs.chromium.org/p/v8/issues/detail?id=1569).

It has been recommended that we list a potential API we could consume in order to create our loader. These extensions are listed below.

### Double Parsing Problem

Due to the fact that we are going to be performing detection of grammar goal using parse failure or the presense of `import`/`export`. It would be ideal if `Compile` and `CompileModule` where merged into a single call. Assuming such we will place our additions in the `CompileOptions` and `Script` types. If such change is not possible, we will be using the fallback method listed in this proposal above.

### API Suggestion

```c++
namespace v8

CompileOptions {
  // parse to Script or Module based upon invalid syntax
  // default is Module
  kDetectGoal
  // tells the parser to change the detect goal default to Script
  kDefaultGoalScript
};

class Module {
  // return a ModuleNamespace view of this Module's exports
  ModuleNamespace Namespace();
}

class SourceTextModule : Script, Module {
  // get a list of imports we need to perform prior to evaluation
  ImportEntry[] ImportEntries();
  
  // get a list of what this exports
  ExportEntry[] ExportEntries();
  
  // cannot be called prior to Run() completing
  ModuleNamespace Namespace();
  
  // required prior to Run()
  //
  // this will add the bindings to the lexical environment of
  // the Module
  FillImports(ImportBinding[] bindings);
}

class DynamicModule : Module {
  // in order for CommonJS modules to create fully formed
  // ES6 Module compatibility we need to hook up a static
  // View of an Object to set as our exports
  //
  // think of this as calling FillImports using the current
  // properties of an object, enumerable or not.
  //
  // exports are never added or removed from the Module even
  // though the exports object may do so, unlike `with()`
  // 
  // construction via this will act as if it has already been
  // run() and fill out the Namespace()
  // this in a way mimics:
  //   1. calling ModuleNamespaceCreate(this, exports)
  //   2. populating the [[Namespace]] field of this Module Record
  DynamicModule(Object exports);
}

class ImportEntry {
  String ModuleRequest();
  
  // note: if ImportName() is "*", the Loader
  // must take the Namespace() and not directly the module
  // as required by ECMA262
  String ImportName();
  
  String LocalName();
}
class ImportBinding {
  ImportBinding(String importLocalName, Module delegate, String delegateExportName);
}
```
