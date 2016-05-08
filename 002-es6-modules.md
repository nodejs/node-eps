| Title  | ES6 Module Interoperability |
|--------|-----------------------------|
| Author | @bmeck                      |
| Status | DRAFT                       |
| Date   | 2016-01-07                  |

**NOTE:** `DRAFT` status does not mean ES6 modules will be implemented in node
core. Instead that this is the standard, should node core decide to implement
ES6 modules. At which time this draft would be moved to `ACCEPTED`.

---

The intent of this standard is to:

* implement interoperability for ES modules and node's existing module system
* create a **[Registry Object][1]** (see WHATWG section below) compatible with
  the [WHATWG Loader](http://whatwg.github.io/loader/) Registry

## Purpose

1. Allow a common module syntax for Browser and Server.
2. Allow a common registry for inspection by Browser and Server
   environments/tools.
    * These will most likely be represented by metaproperties like
      `import.context`, but the spec is not yet fully in place.

## Related


[ECMA262][1] discusses the syntax and semantics of related syntax, and
introduces:

#### Types

* **[ModuleRecord]**
    - Defines the list of imports via [ImportEntry].
    - Defines the list of exports via [ExportEntry].

* **[ModuleNamespace][5]**
    - Represents a read-only static set of live bindings of a module's exports.

#### Operations

* **[ParseModule]**
    - Creates a [SourceTextModuleRecord][1] from source code.

* **[HostResolveImportedModule]**
    - A hook for when an import is exactly performed.

* **[CreateImportBinding]**
    - A means to create a shared binding (variable) from an export to an
      import. Required for the "live" binding of imports.

* **[ModuleNamespaceCreate]**
    - Provides a means of creating a list of exports manually, used so that
      CommonJS `module.exports` can create ModuleRecords that are prepopulated.


[WHATWG Loader](https://github.com/whatwg/loader) discusses the design of
module metadata in a [Registry][1]. All actions regarding the Registry are
synchronous.

**NOTE:** It is not Node's intent to implement the asynchronous pipeline in the
Loader specification. There is discussion about including a synchronous
pipeline in the specification as an addendum.

## Additional Structures Required

### **DynamicModuleRecord**

A Module Record that presents a view of an Object for its `[[Namespace]]`
rather than coming from an environment record.

All exported values are declarative. That means that they are known after the
file is parsed, but before it is evaluated.

When creating a `DynamicModuleRecord` the list of exports is frozen upon
construction. No new exports may be added. No exports may be removed.

## Algorithm

When loading a file:

1. Determine if file is ES or CommonJS (CJS).
2. If CJS
  1. Evaluate immediately
  2. Produce a DynamicModuleRecord from `module.exports`
3. If ES
  1. Parse for `import`/`export`s and keep record, in order to create bindings
  2. Gather all submodules by performing loading dependencies recursively
    * See circular dep semantics below
  3. Connect `import` bindings for all relevant submodules (see
     [ModuleDeclarationInstantiation])
  4. Evaluate

This still guarantees:

* that ES module dependencies are all executed prior to the module itself
* CJS modules have a full shape prior to being handed to ES modules
* allows CJS modules to imperatively start the loading of other modules,
  including ES modules

## Semantics

### Determining if source is an ES Module

A new file type will be recognised, `.mjs` as ES modules. They will be treated
as a different loading semantic but compatible with existing systems, just like
`.node`, `.json`, or usage of `require.extension` (even though deprecated) are
compatible. It would be ideal if we could register the file type with IANA as
an official file type, see [TC39 issue][3]. Though it seems this would need to
go through the [IESG](https://www.ietf.org/iesg/) and it seems browsers are
non-plussed on introducing a new MIME.

The `.mjs` file extension will have a higher loading priority than `.js` for
`require`. This means that, once the node resolution algorithm reaches file
expansion, the path for `path + '.mjs'` would be attempted prior to `path +
'.js'` when performing `require(path)`.

#### Reason for decision

The choice of `.mjs` was made due to a number of factors.

* `.jsm`
    * conflict with Firefox, which includes escalated
      [privileges over the `file://` protocol] that can access
      [sensistive information][4]. This could affect things like
      [test runners providing browser test viewers]
    * [decent usage on npm](https://gist.github.com/ChALkeR/9e1fb15d0a0e18589e6dadc34b80875d)
* `.es`
    * lacks conflicts with other major software
    * removes the JS moniker/signifier in many projects such as Node.js,
      Cylon.js, Three.js, DynJS, JSX, ...
    * removes JavaScript -> JS acronym association for learning
    * is an existing TLD, could be place for squatting / confusion.
    * [small usage on npm](https://gist.github.com/ChALkeR/261550d903ec9867bbab)
* `.m.js`
    * potential conflict with existing software targeting wrong goal
    * allows `*.js` style globbing to work still
    * toolchain problems for asset pipelines/node/editors that only check after
      last `.`
    * [small usage on npm](https://gist.github.com/ChALkeR/c10642f2531b1be36e5d)
* `.mjs`
    * lacks conflicts with other major software, conflicts with
      [metascript](https://github.com/metascript/metascript) but that was last
      published in 2015
    * [small usage on npm](https://gist.github.com/bmeck/07a5beb6541c884acbe908df7b28df3f)

There is knowledge of breakage for code that *upgrades* inner package
dependencies such as `require('foo/bar.js')`. As `bar.js` may move to
`bar.mjs`. Since `bar.js` is not the listed entry point this was considered
okay. If foo was providing this file explicitly like `lodash` has this can be
mitigated easily by using as proxy module should `foo` choose to provide one:

```js
Object.defineProperty(module, 'exports', {
  get() {
    return require(__filename.replace(/\.js$/,'.mjs'))
  },
  configurable:false
});
Object.freeze(module);
```

Concerns of ecosystem damage when using a new file extension were considered as
well. Firewall rules and server scripts using `*.js` as the detection mechanism
for JavaScript would be affected by our change, largely just affecting
browsers. However, new file extensions and mimes continue to be introduced and
supported by such things. If a front-end is unable to make such a change, using
a rewrite rule from `.mjs` to `.js` should sufficiently mitigate this.

There were proposals of using `package.json` as an out of band configuration as
well.

* removes one-off file execution for quick scripting like using `.save` in the
  repl then running.
* causes editors/asset pipelines to have knowledge of this. most work solely on
  file extension and it would be prohibitive to change.
    * Build asset pipelines in particular would be affected such as the Rails
      asset pipeline
    * HTTP/2 PUSH solutions would also need this and would be affected
* no direction for complete removal of CJS. A file extension leads to a world
  without `.js` and only `.mjs` files. This would be a permanent field in
  `package.json`
* per file mode requirements mean using globs or large lists
    * discussion of allowing only one mode per package was a non-starter for
      migration path issues
* `package.json` is not required in `node_modules/` and raw files can live in
  `node_modules/`
    * e.g. `node_modules/foo/index.js`
    * e.g. `node_modules/foo.js`
* complex abstraction to handle symlinks used by tools like
  [link-local](https://github.com/timoxley/linklocal)
    * e.g. `node_modules/foo -> ../app/components/foo.js`

### ES Import Path Resolution

ES `import` statements will not perform non-exact searches on relative or
absolute paths, unlike `require()`. This means that no file extensions, or
index files will be searched for when using relative or absolute paths.

`node_modules` based paths (sometimes called "bare" paths) will continue to use 
searching for both compatibility and to not limit the ability to have
`package.json` support both ES and CJS entry points in a single
codebase.`node_modules` based behavior will continue to be unchanged and look
to parent `node_modules` directories recursively as well. This searching
behavior explicitly includes inner package searching such ass `foo/bar`.

Any entry into `node_modules` via paths not starting with "/", "./", or "../"
will be using the same mechanism of searching while resolving their **entry 
point** as `require()`.

In summary so far:

```javascript
// only looks at
//   ./foo
// does not search:
//   ./foo.js
//   ./foo/index.js
//   ./foo/index.mjs
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
//   /bar/index.mjs
//   /bar/package.json
//   etc.
import '/bar';
```

```javascript
// continues to *search*:
//   ./node_modules/baz.mjs
//   ./node_modules/baz.js
//   ./node_modules/baz/package.json
//   ./node_modules/baz/index.mjs
//   ./node_modules/baz/index.js
// and parent node_modules:
//   ../node_modules/baz.mjs
//   ../node_modules/baz.js
//   ../node_modules/baz/package.json
//   ../node_modules/baz/index.mjs
//   ../node_modules/baz/index.js
//   etc.
import 'baz';
```

```javascript
// continues to *search*:
//   ./node_modules/abc/123.mjs
//   ./node_modules/abc/123.js
//   ./node_modules/abc/123/package.json
//   ./node_modules/abc/123/index.mjs
//   ./node_modules/abc/123/index.js
// and parent node_modules:
//   ../node_modules/abc/123.mjs
//   ../node_modules/abc/123.js
//   ../node_modules/abc/123/package.json
//   ../node_modules/abc/123/index.mjs
//   ../node_modules/abc/123/index.js
//   etc.
import 'abc/123';
```

#### Removal of non-local dependencies

All of the following will not be supported by the `import` statement:

* `$NODE_PATH`
* `$HOME/.node_modules`
* `$HOME/.node_libraries`
* `$PREFIX/lib/node`

Use local dependencies, and symbolic links as needed.

##### How to support non-local dependencies

Although not recommended, and in fact discouraged, there is a way to support
non-local dependencies. **USE THIS AT YOUR OWN DISCRETION**.

Symlinks of `node_modules -> $HOME/.node_modules`, `node_modules/foo/ ->
$HOME/.node_modules/foo/`, etc. will continue to be supported.

Adding a parent directory with `node_modules` symlinked will be an effective
strategy for recreating these functionalities. This will incur the known
problems with non-local dependencies, but now leaves the problems in the hands
of the user, allowing node to give more clear insight to your modules by
reducing complexity.

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

In the case that an `import` statement is unable to find a module, node should
make a **best effort** to see if `require` would have found the module and
print out where it was found, if `NODE_PATH` was used, if `HOME` was used, etc.

#### Vendored modules

This will mean vendored modules are not included in the search path since
`package.json` is not searched for outside of `node_modules`. Please use
[bundledDependencies](https://docs.npmjs.com/files/package.json#bundleddependencies)
to vendor your dependencies instead.

#### Shipping both ES and CJS

Since `node_modules` continues to use searching, when a `package.json` main is
encountered we are still able to perform file extension searches. If we have 2
entry points `index.mjs` and `index.js` by setting `main:"./index"` we can let
Node pick up either, depending on what is supported, without manually needing to
manage multiple entry points separately.

##### Excluding main

Since `main` in `package.json` is entirely optional even inside of npm
packages, some people may prefer to exclude main entirely in the case of using
`./index` as that is still in the node module search algorithm.

### `this` in ES modules

Unlike CJS, ES modules will have a `this` value set to the global scope. This
is a breaking change, CJS modules have a this value set to their `module`
binding.

### ES consuming CommonJS

#### default imports

`module.exports` is a single value. As such it does not have the dictionary
like properties of ES module exports. In order to facilitate named imports for
ES modules, all properties of `module.exports` will be hoisted to named exports
after evaluation of CJS modules with the exception of `default` which will
point to `module.exports` directly.

##### Examples

Given:

```javascript
// cjs.js
module.exports = {
  default:'my-default',
  thing:'stuff'
};
```

You will grab `module.exports` when performing an ES import.

```javascript
// es.mjs

// grabs the namespace
import * as baz from './cjs.js';
// baz = {
//   get default() {return module.exports;},
//   get thing() {return this.default.thing}.bind(baz)
// }

// grabs "default", aka module.exports directly
import foo from './cjs.js';
// foo = {default:'my-default', thing:'stuff'};

// grabs "default", aka module.exports directly
import {default as bar} from './cjs.js';
// bar = {default:'my-default', thing:'stuff'};
```

------

Given:

```javascript
// cjs.js
module.exports = null;
```

You will grab `module.exports` when performing an ES import.

```javascript
// es.mjs
import foo from './cjs.js';
// foo = null;

import * as bar from './cjs.js';
// bar = {default:null};
```

------

Given:

```javascript
// cjs.js
module.exports = function two() {
  return 2;
};
```

You will grab `module.exports` when performing an ES import.

```javascript
// es.mjs
import foo from './cjs.js';
foo(); // 2

import * as bar from './cjs.js';
bar.name; // 'two'
bar.default(); // 2
bar(); // throws, bar is not a function
```

------

Given:

```javascript
// cjs.js
module.exports = Promise.resolve(3);
```

You will grab `module.exports` when performing an ES import.

```javascript
// es.mjs
import foo from './cjs.js';
foo.then(console.log); // outputs 3

import * as bar from './cjs.js';
bar.default.then(console.log); // outputs 3
bar.then(console.log); // throws, bar is not a Promise
```

### CommonJS consuming ES

#### default exports

ES modules only ever declare named exports. A default export just exports a
property named `default`. `require` will not automatically wrap ES modules in a
`Promise`. In the future if top level `await` becomes spec, you can use the
`System.loader.import` function to wrap modules and wait on them (top level
await can cause deadlock with circular dependencies, node should discourage its
use).

##### Examples

Given:

```javascript
// es.mjs
let foo = {bar:'my-default'};
// note:
//   this is a value
//   it is not a binding like `export {foo}`
export default foo;
foo = null;
```

```javascript
// cjs.js
const es_namespace = require('./es');
// es_namespace ~= {
//   get default() {
//     return result_from_evaluating_foo;
//   }
// }
console.log(es_namespace.default);
// {bar:'my-default'}
```

------

Given:

```javascript
// es.mjs
export let foo = {bar:'my-default'};
export {foo as bar};
export function f() {};
export class c {};
```

```javascript
// cjs.js
const es_namespace = require('./es');
// es_namespace ~= {
//   get foo() {return foo;}
//   get bar() {return foo;}
//   get f() {return f;}
//   get c() {return c;}
// }
```

### Known Gotchas

All of these gotchas relate to opt-in semantics and the fact that CommonJS is a
dynamic loader while ES is a static loader.

No existing code will be affected.

#### ES exports are read only

The objects create by an ES module are [ModuleNamespace Objects][5].

These have `[[Set]]` be a no-op and are read only views of the exports of an ES
module. Attempting to reassign any named export will not work, but assigning to
the properties of the exports follows normal rules.

### CJS exports allow mutation

Unlike ES modules, CJS modules have allowed mutation. When ES modules are
integrating against CJS systems like Grunt, it may be necessary to mutate a
`module.exports`.

Remember that `module.exports` from CJS is directly available under `default`
for `import`. This means that if you use:

```javascript
import * as namespace from 'grunt';
```

According to ES `*` grabs the namespace directly whose properties will be
read-only.

However, doing:

```javascript
import grunt_default from 'grunt';
```

Grabs the `default` which is exactly what `module.exports` is, and all the
properties will be mutable.

#### ES will not honor reassigning `module.exports` after evaluation

Since we need a consistent time to snapshot the `module.exports` of a CJS
module. We will execute it immediately after evaluation. Code such as:

```javascript
// bad-cjs.js
module.exports = 123;
setTimeout(_ => module.exports = null);
```

Will not see `module.exports` change to `null`. All ES module `import`s of the
module will always see `123`.

#### ES export list for CJS are snapshot immediately after execution.

Since `module.exports` is snapshot immediately after execution, that is the
point when hoisting of properties occurs, adding and removing properties later
will not have an effect on the list of exports.

```javascript
// bad-cjs.js
module.exports = {
  yo: 'lo'
};
setTimeout(_ => {
  delete module.exports.yo;
  module.exports.foo = 'bar';
  require('./es.js');
});
```

```javascript
// es.js
import * as namespace from './bad-cjs.js';
console.log(Object.keys(namespace)); // ['yo']
console.log(namespace.foo); // undefined

// mutate to show 'yo' still exists as a binding
import cjs_exports from './bad-cjs.js';
cjs_exports.yo = 'lo again';
console.log(namespace.yo); // 'yolo again'
```

#### Circular Dep CJS => ES => CJS Causes Throw

Due to the following explanation we want to avoid a very specific problem.
Given:

```javascript
// cjs.js
module.exports = {x:0};
require('./es');
```

```javascript
// es.mjs
import * as ns from './cjs.js';
// ns = ?
import cjs from './cjs.js';
// cjs = ?
```

ES modules must know the list of exported bindings of all dependencies prior to
evaluating. The value being exported for CJS modules is not stable to snapshot
until `cjs.js` finishes executing. The result is that there are no properties
to import and you receive an empty module.

In order to prevent this sticky situation we will throw on this case.

Since this case is coming from ES, we will not break any existing circular
dependencies in CJS <-> CJS. It may be easier to think of this change as
similar to how you cannot affect a generator while it is running.

This would change the ES module behavior to:

```javascript
// es.mjs
import * as ns from './cjs.js';
// throw new EvalError('./cjs is not an ES module and has not finished evaluation');
```

## Advisory

V8 currently does not expose the proper APIs for creating Loaders, it has done
the brunt of the work by [exposing a parser in their issue tracker].

It has been recommended that we list a potential API we could consume in order
to create our loader. These extensions are listed below.

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
  ModuleDeclarationInstantiation(ImportBinding[] bindings);
}

class DynamicModule : Module {
  // in order for CommonJS modules to create fully formed
  // ES Module compatibility we need to hook up a static
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
  return Object.freeze(module_namespace);
}
```

## Example Implementation

These are written with the expectation that:

* ModuleNamespaces can be created from existing Objects.
* WHATWG Loader spec Registry is available as a ModuleRegistry.
* ModuleStatus Objects can be created.

The variable names should be hidden from user code using various techniques
left out here.

### CJS Modules

#### Pre Evaluation

```javascript
// for posterity, will still throw on circular deps
ModuleRegistry.set(__filename, new ModuleStatus({
    'ready': {'[[Result]]':undefined};
}));
```

#### Immediately Post Evaluation

##### On Error

```javascript
ModuleRegistry.delete(__filename);
```

##### On Normal Completion

```javascript
let module_namespace = Object.create(module.exports);
Object.defineProperty(module_namespace, 'default', {
    value: module.exports,
    writable: false,
    configurable: false
});
ModuleRegistry.set(__filename, new ModuleStatus({
    'ready': {'[[Result]]':v8.module.DynamicModule(module_namespace)};
}));
```

### ES Modules

#### Post Parsing

```javascript
Object.defineProperty(module, 'exports', {
  get() {return v8.Module.Namespace(module)};
  set(v) {throw new Error(`${__filename} is an ES module and cannot assign to module.exports`)}
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


[1]: https://tc39.github.io/ecma262/#sec-source-text-module-records
[2]: https://whatwg.github.io/loader/#registry "Registry Objects"
[3]: https://github.com/tc39/ecma262/issues/322 "Add application/javascript+module mime to remove ambiguity"
[4]: https://developer.mozilla.org/en-US/docs/Mozilla/JavaScript_code_modules/Services.jsm "Services.jsm"
[5]: https://tc39.github.io/ecma262/#sec-module-namespace-exotic-objects "Module Namespace Exotic Objects"

[ModuleRecord]: https://tc39.github.io/ecma262/#sec-abstract-module-records "Abstract Module Records"
[ImportEntry]: https://tc39.github.io/ecma262/#table-39 "ImportEntry Record Fields"
[ExportEntry]: https://tc39.github.io/ecma262/#table-41 "ExportEntry Record Fields"
[ParseModule]: https://tc39.github.io/ecma262/#sec-parsemodule "ParseModule"
[HostResolveImportedModule]: https://tc39.github.io/ecma262/#sec-hostresolveimportedmodule "Runtime Semantics: HostResolveImportedModule"
[CreateImportBinding]: https://tc39.github.io/ecma262/#sec-createimportbinding "CreateImportBinding"
[ModuleNamespaceCreate]: https://tc39.github.io/ecma262/#sec-modulenamespacecreate "ModuleNamespaceCreate"
[ModuleDeclarationInstantiation]: https://tc39.github.io/ecma262/#sec-moduledeclarationinstantiation "ModuleDeclarationInstantiation() Concrete Method"
[test runners providing browser test viewers]: https://mochajs.org/#running-mocha-in-the-browser "Running Mocha in the Browser"
[exposing a parser in their issue tracker]: https://bugs.chromium.org/p/v8/issues/detail?id=1569 "Implement Harmony Modules"
[privileges over the `file://` protocol]: https://developer.mozilla.org/en-US/docs/Mozilla/JavaScript_code_modules/Using#The_URL_for_a_code_module "The URL for a code module"
