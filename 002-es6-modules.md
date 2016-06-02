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

* **[ModuleNamespace]
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

A `ModuleNamespaceObject` that performs delegation to an Object when accessing
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

The abstract operation DelegatedModuleNamespaceObjectCreate with arguments `O`
is used to create a DelegatedModuleNamespaceObject. It performs the following
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

The abstractjob operation NodeModuleEvaluationJob with parameters `source` and
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

* that ES module dependencies are all executed prior to the module itself
* CJS modules have a full shape prior to being handed to ES modules
* allows CJS modules to imperatively start the loading of other modules,
  including ES modules

## 5. Semantics

### 5.1. Determining if source is an ES Module

A new file type will be recognised, `.mjs` as ES modules. `.mjs` files will be
treated as having different loading semantics that are compatible with the
existing CJS system, just like `.node`, `.json`, or usage of
`require.extension` (even though deprecated) are compatible. This file type
will be registered with IANA as an official file type, see [TC39 issue]
(https://github.com/tc39/ecma262/issues/322).

The `.mjs` file extension will have a higher loading priority than `.js` for
`require`. This means that, once the Node resolution algorithm reaches file
expansion, the path for `path + '.mjs'` would be attempted prior to `path +
'.js'` when performing `require(path)`.

#### 5.1.1 Reason for decision

The choice of `.mjs` was made due to a number of factors.

* `.jsm`
    * conflict with Firefox, which includes escalated
      [privileges over the `file://` protocol] that can access
      [sensistive information][4]. This could affect things like
      [test runners providing browser test viewers]
    * [decent usage on npm]
      (https://gist.github.com/ChALkeR/9e1fb15d0a0e18589e6dadc34b80875d)
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
    * toolchain problems for asset pipelines/Node/editors that only check after
      last `.`
    * [small usage on npm](https://gist.github.com/ChALkeR/c10642f2531b1be36e5d)
* `.mjs`
    * lacks conflicts with other major software, conflicts with
      [metascript](https://github.com/metascript/metascript) but that was last
      published in 2015
    * [small usage on npm]
      (https://gist.github.com/bmeck/07a5beb6541c884acbe908df7b28df3f)


#### 5.1.1.1 Inter package loading using file extension  breakage.

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

It is recommended going forward that developers not rely on the file extensions
in packages they do not control.

### 5.1.2. Ecosystem Concerns

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

### 5.2. ES Import Path Resolution

ES `import` statements will perform non-exact searches on relative or
absolute paths, like `require()`. This means that file extensions, and
index files will be searched,

In summary:

```javascript
// looks at
//   ./foo.mjs
//   ./foo.js
//   ./foo/package.json
//   ./foo/index.mjs
//   ./foo/index.js
//   etc.
import './foo';
```

```javascript
// looks at
//   /bar.mjs
//   /bar.js
//   /bar/package.json
//   /bar/index.mjs
//   /bar/index.js
//   etc.
import '/bar';
```

```javascript
// looks at:
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
// looks at:
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

#### 5.2.3. Shipping both ES and CJS

When a `package.json` main is encountered, file extension searches are used to
provide a means to ship both ES and CJS variants of packages. If we have two
entry points `index.mjs` and `index.js` setting `"main":"./index"` in
`package.json` will make Node pick up either, depending on what is supported.

##### 5.2.3.1. Excluding main

Since `main` in `package.json` is entirely optional even inside of npm
packages, some people may prefer to exclude main entirely in the case of using
`./index` as that is still in the Node module search algorithm.

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

### 5.5. CommonJS consuming ES

#### 5.5.1. default exports

ES modules only export named values. A "default" export is an export that uses
the property named `default`.

##### 5.5.1.1. Examples

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

### 5.6. Known Gotchas

All of these gotchas relate to opt-in semantics and the fact that CommonJS is a
dynamic loader while ES is a static loader.

No existing code will be affected.

#### 5.6.1. ES exports are read only

The objects create by an ES module are [ModuleNamespace Objects][5].

These have `[[Set]]` be a no-op and are read only views of the exports of an ES
module. Attempting to reassign any named export will not work, but assigning to
the properties of the exports follows normal rules.

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
cjs_exports.yo = 'lo again';
console.log(namespace.yo); // 'yolo again'
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

## 6. Example Implementations

These are written with the expectation that:

* ModuleNamespaces can be created from existing Objects.
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
