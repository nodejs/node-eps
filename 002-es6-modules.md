| Title  | ES6 Module Interoperability |
|--------|-----------------------------|
| Author | @bmeck                      |
| Status | DRAFT                       |
| Date   | 2016-01-07                  |

**NOTE:** `DRAFT` status does not mean ES6 modules will be implemented in Node
core. Instead that this is the standard, should Node core decide to implement
ES6 modules. At which time this draft would be moved to `ACCEPTED`.

---

The intent of this standard is to:

* implement interoperability for ES modules and Node's existing module system
* create a **Registry Object** (see WHATWG section below) compatible with
  the [WHATWG Loader](http://whatwg.github.io/loader/) Registry

## 1. Purpose

1. Allow a common module syntax for Browser and Server.
2. Allow a common registry for inspection by Browser and Server
   environments/tools.
    * These will most likely be represented by metaproperties like
      `import.context`, but the spec is not yet fully in place.

## 2. Related


[ECMA262](tc39.github.io/ecma262/) discusses the syntax and semantics of
related syntax, and introduces:

### 2.1. Types

* **[ModuleRecord]
(https://tc39.github.io/ecma262/#sec-abstract-module-records)**
    - Defines the list of imports via `[[ImportEntry]]`.
    - Defines the list of exports via `[[ExportEntry]]`.

* **[Module Namespace]
(https://tc39.github.io/ecma262/#sec-module-namespace-objects)**
    - Represents a read-only static set of bindings to a module's exports.

### 2.2. Operations

* **[ParseModule](https://tc39.github.io/ecma262/#sec-parsemodule)**
    - Creates a [SourceTextModuleRecord]
    (https://tc39.github.io/ecma262/#sec-source-text-module-records) from
    source code.

* **[HostResolveImportedModule]
(https://tc39.github.io/ecma262/#sec-hostresolveimportedmodule)**
    - A hook for when an import is exactly performed. This returns a
      `ModuleRecord`. Used as a means to grab modules from Node's loader/cache.

* **[CreateImportBinding]
(https://tc39.github.io/ecma262/#sec-createimportbinding)**
    - A means to create a shared binding (variable) from an export to an
      import. Required for the "live" binding of imports.

* **[ModuleNamespaceCreate]
(https://tc39.github.io/ecma262/#sec-modulenamespacecreate)**
    - Provides a means of creating a list of exports manually, used so that
      CommonJS `module.exports` can create `ModuleRecord`s that are
      prepopulated.


[WHATWG Loader](https://github.com/whatwg/loader) discusses the design of
module metadata in a [Registry](https://whatwg.github.io/loader/#registry). All
actions regarding the Registry can be done
synchronously, though JS level API uses Promises.

**NOTE:** It is not Node's intent to implement the asynchronous pipeline in the
WHATWG Loader specification.

## 3. Additional Structures Required

### 3.1. **DynamicModuleRecord**

A Module Record that represents a view of an Object for its `[[Namespace]]`
rather than coming from an environment record.

`DynamicModuleRecord` preserves the feature that exported values are known when
it comes time for `HostResolveImportedModule` to return. That means that they
are known after the
file is parsed, but before it is evaluated. This behavior is preserved by Node
synchronously executing CJS files when they are encountered during
`HostResolveImportedModule`.

When creating a `DynamicModuleRecord` the [`[[Exports]]`]
(https://tc39.github.io/ecma262/#table-29) is frozen upon construction. No new
exports may be added. No exports may be removed. The values of the exports will
continue to be mutable however.

### 3.1.1 DynamicModuleCreate(O)

The abstract operation `DynamicModuleCreate` with arguments `namespace` is used
to allow creation of new `DynamicModuleRecord`s. It performs the following
steps:

1. Let M be a newly created object.
2. Set M's essential internal methods to the definitions specified in
   [15.2.1.15 Abstract Module Records]
   (https://tc39.github.io/ecma262/#sec-abstract-module-records)
3. Set M's [[Realm]] internal slot to the current Realm Record.
4. Set M's [[Namespace]] internal slot to DelegatedModuleNamespaceObjectCreate
   (`M`, `O`)
5. Set M's [[Environment]] internal slot to NewObjectEnvironment(`M`.[[Namespace]], **null**)
6. Set M's [[Evaluated]] internal slot to **true**
7. Return M

### 3.2. **DelegatedModuleNamespaceObject**

A Module Namespace Object that performs delegation to an Object when accessing
properties. This is used for delegation behavior from CJS `module.exports` when
imported by ES modules.

#### Table 1: Internal Slots of DelegatedModuleNamespaceObject Namespace Exotic
Objects

Field Name | Value Type | Meaning
---| --- | ---
[[Delegate]] | Object  | The Object from which to delegate access

#### 3.2.1. `[[Get]] (P, Receiver)`

When the [[Get]] internal method of a module namespace exotic object O is
called with property key P and ECMAScript language value Receiver, the
following steps are taken:

1. Assert: IsPropertyKey(`P`) is true.
2. If Type(`P`) is Symbol, then
3. Return ? OrdinaryGet(`O`,`P`,` Receiver`).
4. Let exports be the value of `O`'s `[[Exports]]` internal slot.
5. If `P` is not an element of exports, return **undefined**.
6. Let m be the value of `O`'s `[[Object]]` internal slot.
7. If `P` equals **"default"**, return `m`.
7. Let `value` be ! `O`.`[[Get]]`(`P`,` O`)
8. Return `value`

#### 3.2.2. `DelegatedModuleNamespaceObjectCreate(module, O)`

The abstract operation `DelegatedModuleNamespaceObjectCreate` with arguments `O`
is used to create a `DelegatedModuleNamespaceObject`. It performs the following
steps:

1. Assert: `module` is a Module Record.
2. Assert: `module`.[[Namespace]] is **undefined**.
3. Let `NS` be a newly created object.
4. Set `NS`'s essential internal methods to the definitions specified in [9.4.6
   Module Namespace Exotic Objects]
   (https://tc39.github.io/ecma262/#sec-module-namespace-exotic-objects)
5. Set `NS`'s [[Module]] internal slot to `module`
6. Let `exports` be a new List
7. Let `p` be `O`.
8. Let `done` be **false**.
9. Repeat while `done` is **false**,
    1. If `p` is null, let `done `be **true**.
    2. Else,
        1. For each `property` in OwnPropertyKeys(`p`)
            1. If `property` does not equal **"default"** and `exports` does
               not contain `property`, add `property` to exports
        2. If the [[GetPrototypeOf]] internal method of `p` is not the ordinary
           object internal method defined in [9.1.1]
           (https://tc39.github.io/ecma262/#sec-ordinary-object-internal-methods-and-internal-slots-getprototypeof)
           , let `done` be **true**.
        3. Else, let `p` be the value of p's [[Prototype]] internal slot.
10. Set the value of the [[Exports]] internal slot of `NS` to `exports`.
11. Return `NS`

## 4. Algorithm

### 4.1. NodeModuleEvaluationJob(source, mode)

The abstractjob operation `NodeModuleEvaluationJob` with parameters `source` and
`mode`. The results of this should be placed in the cache that
`HostResolveImportedModule` uses.

1. If `mode` equals CJS
    1. Let `body` be the bootstraped form of `source` with necessary CJS
       wrapper code.
    2. Call ! ScriptEvaluationJob(`body`, **undefined**)
    3. Return ! DynamicModuleCreate from `module.exports`
2. Else if `mode` equals ES
    1. Let `M` be ! ParseModule(`source`)
    2. Perform the algorithm listed in **4.** on `M`.[[RequestedModules]] while
       respecting the semantics in **5.**
    3. Connect `import` bindings for all relevant submodules using
     [ModuleDeclarationInstantiation from the Abstract Module Record M]
     (https://tc39.github.io/ecma262/#table-37)
    4. Call ! `M`.[[ModuleEvaluation]]
    5. Return `M`

**NOTE:** This still guarantees:

* ES module dependencies are all executed prior to the module itself
* CJS modules have a full shape prior to being handed to ES modules
* CJS modules can imperatively start loading other modules, including ES modules

## 5. Semantics

### 5.1. Determining if source is an ES Module

Require that Module source text has at least one `import` or `export` declaration.
A module with only an `import` declaration and no `export` declaration is valid.
Modules, that do not export anything, should specify an `export {}` to make
intentions clear and avoid accidental parse errors while removing `import`
declarations. The `export {}` is **not** new syntax and does **not** export an
empty object. It is simply the standard way to specify exporting nothing.

A package opts-in to the Module goal by specifying `"module"` as the parse goal
field *(name not final)* in its `package.json`. Package dependencies are not
affected by the opt-in and may be a mix of CJS and ES module packages. If a parse
goal is not specified, then attempt to parse source text as the preferred goal
*(Script for now since most modules are CJS)*. If there is a parse error that
may allow another goal to parse, then parse as the other goal, and so on. After
this, the goal is known unambiguously and the environment can safely perform
initialization without the possibility of the source text being run in the wrong
goal.

<em>Note: While the ES2015 specification
[does not forbid](http://www.ecma-international.org/ecma-262/6.0/#sec-forbidden-extensions)
this extension, Node wants to avoid acting as a rogue agent. Node has a TC39
representative, [@bmeck](https://github.com/bmeck), to champion this proposal.
A specification change or at least an official endorsement of this Node proposal
would be welcomed. If a resolution is not possible, this proposal will fallback
to the previous [`.mjs` file extension proposal](https://github.com/nodejs/node-eps/blob/5dae5a537c2d56fbaf23aaf2ae9da15e74474021/002-es6-modules.md#51-determining-if-source-is-an-es-module).</em>

#### 5.1.1 Goal Detection

##### Parse (source, goal, throws)

  The abstract operation to parse source text as a given goal.

  1. Bootstrap `source` for `goal`.
  2. Parse `source` as `goal`.
  3. If success, return `true`.
  4. If `throws`, throw exception.
  5. Return `false`.

##### Operation

1. If a package parse goal is specified, then
  1. Let `goal` be the resolved parse goal.
  2. Call `Parse(source, goal, true)` and return.

2. Else fallback to multiple parse.
  1. If `Parse(Source, Script, false)` is `true`, then
    1. Return.
  2. Else
    1. Call `Parse(Source, Module, true)`.

  *Note: A host can choose either goal to parse first and may change their order
  over time or as new parse goals are introduced. Feel free to swap the order of
  Script and Module.*

#### 5.1.2 Implementation

To improve performance, host environments may want to specify a goal to parse
first. This can be done in several ways:<br>
cache on disk, a command line flag, a manifest file, HTTP header, file extension, etc.

#### 5.1.3 Tooling Concerns

Some tools, outside of Node, may not have access to a JS parser *(Bash programs,
some asset pipelines, etc.)*. These tools generally operate on files as opaque
blobs / plain text files and can use the techniques, listed under
[Implementation](#512-implementation), to get parse goal information.

### 5.2. ES Import Path Resolution

ES `import` statements will perform non-exact searches on relative or
absolute paths, like `require()`. This means that file extensions, and
index files will be searched,

In summary:

```javascript
// looks at
//   ./foo.js
//   ./foo/package.json
//   ./foo/index.js
//   etc.
import './foo';
```

```javascript
// looks at
//   /bar.js
//   /bar/package.json
//   /bar/index.js
//   etc.
import '/bar';
```

```javascript
// looks at:
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

```javascript
// looks at:
//   ./node_modules/abc/123.js
//   ./node_modules/abc/123/package.json
//   ./node_modules/abc/123/index.js
// and parent node_modules:
//   ../node_modules/abc/123.js
//   ../node_modules/abc/123/package.json
//   ../node_modules/abc/123/index.js
//   etc.
import 'abc/123';
```

#### 5.2.1. Removal of non-local dependencies

All of the following will not be supported by the `import` statement:

* `$NODE_PATH`
* `$HOME/.node_modules`
* `$HOME/.node_libraries`
* `$PREFIX/lib/node`

Use local dependencies, and symbolic links as needed.

##### 5.2.1.1. How to support non-local dependencies

Although not recommended, and in fact discouraged, there is a way to support
non-local dependencies. **USE THIS AT YOUR OWN DISCRETION**.

Symlinks of `node_modules -> $HOME/.node_modules`, `node_modules/foo/ ->
$HOME/.node_modules/foo/`, etc. will continue to be supported.

Adding a parent directory with `node_modules` symlinked will be an effective
strategy for recreating these functionalities. This will incur the known
problems with non-local dependencies, but now leaves the problems in the hands
of the user, allowing Node to give more clear insight to your modules by
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

#### 5.2.2. Errors from new path behavior.

In the case that an `import` statement is unable to find a module, Node should
make a **best effort** to see if `require` would have found the module and
print out where it was found, if `NODE_PATH` was used, if `HOME` was used, etc.

### 5.3. `this` in ES modules

ES modules will have a `this` value set to `undefined`. This
is a breaking change. CJS modules have a `this` value set to their `module`
binding.

See ECMA262's [Module Environment Record](
https://tc39.github.io/ecma262/#sec-module-environment-records-getthisbinding
) for this semantic.

### 5.4. ES consuming CommonJS

####5.4.1. default imports

`module.exports` is a single value. As such it does not have the dictionary
like properties of ES module exports. In order to facilitate named imports for
ES modules, all properties of `module.exports` will be hoisted to named exports
after evaluation of CJS modules with the exception of `default` which will
point to `module.exports` directly.

##### 5.4.1.1. Examples

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
// es.js

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
// es.js
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
// es.js
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
// es.js
import foo from './cjs.js';
foo.then(console.log); // outputs 3

import * as bar from './cjs.js';
bar.default.then(console.log); // outputs 3
bar.then(console.log); // throws, bar is not a Promise
```

### 5.5. CommonJS consuming ES

#### 5.5.1. default exports

ES modules only export named values. A "default" export is an export that uses
the property named `default`.

##### 5.5.1.1. Examples

Given:

```javascript
// es.js
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
// es.js
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

### 5.6. Known Gotchas

All of these gotchas relate to opt-in semantics and the fact that CommonJS is a
dynamic loader while ES is a static loader.

No existing code will be affected.

#### 5.6.1. ES exports are read only

When CommonJS consumes ES, `require()` returns a [Module Namespace Object](https://tc39.github.io/ecma262/#sec-module-namespace-objects). These have a no-op `[[Set]]` that makes them read only views of the exports of an ES module. Attempting to assign to any property of the Module Namespace Object will not work, but assignment to properties of exported objects follows normal rules. Example:

```javascript
const es_namespace = require('./es');

// Doesn't work:
es_namespace.foo = {};

// Normal rules:
es_namespace.foo.bar = {};
```

### 5.7. CJS modules allow mutation of imported modules

CJS modules have allowed mutation on imported modules. When ES modules are
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

#### 5.7.1. ES will not honor reassigning `module.exports` after evaluation

Since we need a consistent time to snapshot the `module.exports` of a CJS
module. We will execute it immediately after evaluation. Code such as:

```javascript
// bad-cjs.js
module.exports = 123;
setTimeout(_ => module.exports = null);
```

Will not see `module.exports` change to `null`. All ES module `import`s of the
module will always see `123`.

#### 5.7.2. ES export list for CJS are snapshot immediately after execution.

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
cjs_exports.yo += ' again';
console.log(namespace.yo); // 'lo again'
```

#### 5.7.3. Circular Dep CJS => ES => CJS Causes Throw

Due to the following explanation we want to avoid a very specific problem.
Given:

```javascript
// cjs.js
module.exports = {x:0};
require('./es');
```

```javascript
// es.js
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
// es.js
import * as ns from './cjs.js';
// throw new EvalError('./cjs is not an ES module and has not finished evaluation');
```

## 6. Example Implementations

These are written with the expectation that:

* Module Namespace Objects can be created from existing Objects.
* WHATWG Loader spec Registry is available as a ModuleRegistry.
* ModuleStatus Objects can be created.

The variable names should be hidden from user code using various techniques
left out here.

### 6.1. CJS Modules

#### 6.1.1. Pre Evaluation

```javascript
// for posterity, will still throw on circular deps
ModuleRegistry.set(__filename, new ModuleStatus({
    'ready': {'[[Result]]':undefined};
}));
```

#### 6.1.2. Immediately Post Evaluation

##### 6.1.2.1. On Error

```javascript
ModuleRegistry.delete(__filename);
```

##### 6.1.2.2. On Normal Completion

```javascript
let module_namespace = module.exports;
ModuleRegistry.set(__filename, new ModuleStatus({
    'ready': {'[[Result]]':DynamicModuleCreate(module_namespace)[[Namespace]]};
}));
```

### 6.2. ES Modules

#### 6.2.1. Post Parsing

```javascript
Object.defineProperty(module, 'exports', {
  get() {return module[[Namespace]]};
  set(v) {throw new Error(`${__filename} is an ES module and cannot assign to module.exports`)}
  configurable: false,
  enumerable: false
});
```

Parsing occurs prior to evaluation, and CJS may execute once we start to
resolve `import`.

#### 6.2.2. Header

```javascript
// we will intercept this to inject the values
import {__filename,__dirname,require,module,exports} from '';
```

#### 6.2.3. Immediately Post Evaluation

##### 6.2.3.1. On Error

```javascript
delete require.cache[__filename];
```
