# ES6 Modules

Status | Draft
------ | ----
Author | Bradley Meck
Date   | 7 Jan 2016

It is our intent to:

* implement interoperability for ES6 modules and node's existing module system
* create a **Registry Object** (see WHATWG section below) compatible with the WHATWG Loader Registry

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

### [WHATWG Loader](https://github.com/whatwg/loader)

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

A new filetype will be recognised, `.jsm` as ES6 based modules. They will be treated as a different loading semantic but compatible with existing systems, just like `.node`, `.json`, or usage of `require.extension` (even though deprecated) are compatible. It would be ideal if we could register the filetype with IANA as an offical file type, see [TC39 issue](https://github.com/tc39/ecma262/issues/322). Though it seems this would need to go through the [IESG](https://www.ietf.org/iesg/) and it seems browsers are non-plussed on introducing a new MIME.

The `.jsm` file extension will have a higher loading priority than `.js`.

### ES6 Import Resolution

ES6 `import` statements will not perform non-exact searches on relative or absolute paths, unlike `require()`. This means that no file extensions, or index files will be searched for when using relative or absolute paths. 

`node_modules` based paths will continue to use searching for both compatibility and to not limit the ability to have `package.json` support both ES6 and CJS entry points in a single codebase. `node_modules` based behavior will continue to be unchanged and look to parent `node_modules` directories recursively as well.

In summary so far:

```javascript
// only looks at
//   ./foo
// does not search:
//   ./foo.js
//   ./foo/index.js
//   ./foo/package.json
//   etc.
import './foo';
```

```javascript
// only looks at
//   /bar
// does not search:
//   /bar.js
//   /bar/index.js
//   /bar/package.json
//   etc.
import '/bar';
```

```javascript
// continues to *search*:
//   ./node_modules/baz.js
//   ./node_modules/baz/package.json
//   ./node_modules/baz/index.js
// and parent node_modules:
//   ../node_modules/baz.js
//   ../node_modules/baz/package.json
//   ../node_modules/baz/index.js
//   etc.
import 'baz';
```

#### Removal of non-local dependencies

All of the following:

* `$NODE_PATH`
* `$HOME/.node_modules`
* `$HOME/.node_libraries`
* `$PREFIX/lib/node`

will not be supported by the `import` statement. Use local dependencies, and symbolic links as needed.

##### How to support non-local dependencies

Although not recommended, and in fact discouraged, there is a way to support non-local dependencies. **USE THIS AT YOUR OWN DISCRETION**. 

Symlinks of `node_modules -> $HOME/.node_modules`, `node_modules/foo/ -> $HOME/.node_modules/foo/`, etc. will continue to be supported.

Adding a parent directory with `node_modules` symlinked will be an effective strategy for recreating these functionalities. This will incur the known problems with non-local dependencies, but now leaves the problems in the hands of the user, allowing node to give more clear insight to your modules by reducing complexity.

Given:

```sh
/opt/local/myapp
```

Transform to:

```sh
/opt/local/non-local-deps/myapp
/opt/local/non-local-deps/node_modules -> $PREFIX/lib/node (etc.)
```

And nest as many times as needed.

#### Errors

In the case that an `import` statement is unable to find a module, node should make a **best effort** to see if `require` would have found the module and print out where it was found, if `NODE_PATH` was used, if `HOME` was used, etc.

#### Vendored modules

This will mean vendored modules are not included in the search path since `package.json` is not searched for outside of `node_modules`. Please use [bundledDependencies](https://docs.npmjs.com/files/package.json#bundleddependencies) to vendor your dependencies instead.

#### Shipping both ES6 and CJS

Since `node_modules` continues to use searching, when a `package.json` main is encountered we are still able to perform file extension searches. If we have 2 entry points `index.jsm` and `index.js` by setting `main:"./index"` we can let Node pick up either depending on what is supported, without us needing to manage multiple entry points separately.

### CommonJS consuming ES6

#### default exports

ES6 modules only ever declare named exports. A default export just exports a property named `default`. `require` will not automatically wrap ES6 modules in a `Promise`. In the future if top level `await` becomes spec, you can use the `System.loader.import` function to wrap modules and wait on them (top level await can cause deadlock with circular dependencies, node should discourage its use).

Given

```javascript
let foo = 'my-default';
export default foo;
```

```javascript
const foo = require('./es6').foo;
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
// es6.jsm
import foo from './cjs';
// foo = {default:'my-default'};

import {default as bar} from './cjs';
// bar = {default:'my-default'};
```

### Known Gotchas

All of these gotchas relate to opt-in semantics and the fact that CommonJS is a dynamic loader while ES6 is a static loader.

No existing code will be affected.

#### Circular Dep CJS => ES6 => CJS Causes Throw

Due to the following explanation we want to avoid a very specific problem.  Given:

```javascript
//cjs.js
module.exports = {x:0};
require('./es6');
```

```javascript
//es6.jsm
import * as ns from './cjs';
// ns = undefined
import cjs from './cjs';
// ns = undefined
```

The list of properties being exported is not populated until `cjs.js` finishes executing. The result is that there are no properties to import and you recieve an empty module.

In order to prevent this sticky situation we will throw on this case.

Since this case is coming from ES6, we will not break any existing circular dependencies in CJS <-> CJS. It may be easier to think of this change as similar to how you cannot affect a generator while it is running.

This would change the ES6 module behavior to:

```javascript
//es6.jsm
import * as ns from './cjs';
// throw new EvalError('./cjs is not an ES6 module and has not finished evaluation');
```

## Advisory

V8 currently does not expose the proper APIs for creating Loaders, it has done the brunt of the work by [exposing a parser in their issue tracker](https://bugs.chromium.org/p/v8/issues/detail?id=1569).

It has been recommended that we list a potential API we could consume in order to create our loader. These extensions are listed below.

### API Suggestion

```cpp
namespace v8;

class Module {
  // return a ModuleNamespace view of this Module's exports
  ModuleNamespace Namespace();
}

class SourceTextModule : Script, Module {
  // get a list of imports we need to perform prior to evaluation
  ImportEntry[] ImportEntries();
  
  // get a list of what this exports
  ExportEntry[] ExportEntries();
  
  // can be called prior to Run(), but all entries will have values of undefined
  ModuleNamespace Namespace();
  
  // required prior to Run()
  //
  // this will add the bindings to the lexical environment of
  // the Module
  ImportDeclarationInstantiation(ImportBinding[] bindings);
}

class DynamicModule : Module {
  // in order for CommonJS modules to create fully formed
  // ES6 Module compatibility we need to hook up a static
  // View of an Object to set as our exports
  //
  // think of this as calling ImportDeclarationInstantiation using the current
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
  // 
  // see JS implementation below for approximate behavior
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

```javascript
// JS implementation of DynamicModule
function DynamicModule(obj) {
  let module_namespace = Object.create(null);
  function gatherExports(obj, acc = new Set()) {
      if (typeof obj !== 'object' && typeof obj !== 'function' || obj === null) {
          return acc;
      }
      for (const key of Object.getOwnPropertyNames(obj)) {
          const desc = Object.getOwnPropertyDescriptor(obj, key);
          acc.add({key,desc});
      }
      return gatherExports(Object.getPrototypeOf(obj), acc);
  }
  [...gatherExports(obj)].forEach(({key,desc}) => {
      if (key === 'default') return;
      Object.defineProperty(module_namespace, key, {
          get: () => obj[key],
          set() {throw new Error(`ModuleNamespace key ${key} is read only.`)},
          configurable: false,
          enumerable: Boolean(desc.enumerable)
      });
  });
  return module_namespace;
}
```

## Example Implementation

These are written with the expectation that:

* ModuleNamespaces can be created from existing Objects.
* WHATWG Loader spec Registry is available as ES6ModuleRegistry.
* ModuleStatus Objects can be created.

The variable names should be hidden from user code using various techniques left out here.

### CJS Modules

#### Pre Evaluation

```javascript
// for posterity, will still throw on circular deps
ES6ModuleRegistry.set(__filename, new ModuleStatus({
    'ready': {'[[Result]]':undefined};
}));
```

#### Immediately Post Evaluation

##### On Error

```javascript
ES6ModuleRegistry.delete(__filename);
```

##### On Normal Completion

```javascript
let module_namespace = Object.create(module.exports);
Object.defineProperty(module_namespace, 'default', {
    value: module.exports,
    writable: false,
    configurable: false
});
ES6ModuleRegistry.set(__filename, new ModuleStatus({
    'ready': {'[[Result]]':v8.module.DynamicModule(module_namespace)};
}));
```

### ES6 Modules

#### Post Parsing

```javascript
Object.defineProperty(module, 'exports', {
  get() {return v8.Module.Namespace(module)};
  set(v) {throw new Error(`${__filename} is an ES6 module and cannot assign to module.exports`)}
  configurable: false,
  enumerable: false
});
```

Parsing occurs prior to evaluation, and CJS may execute once we start to resolve `import`.

#### Header

```javascript
// we will intercept this to inject the values
import {__filename,__dirname,require,module,exports} from 'CURRENT__FILENAME';
// to prevent global problems, and false sense of writable exports object:
// exports = undefined
```

#### Immediately Post Evaluation

##### On Error

```javascript
delete require.cache[__filename];
```
