| Title  | ESM Interoperability |
|--------|-----------------------------|
| Author | @bmeck                      |
| Status | DRAFT                       |
| Date   | 2016-01-07                  |

**NOTE:** `DRAFT` status does not mean ESM will be implemented in Node
core. Instead that this is the standard, should Node core decide to implement
ESM. At which time this draft would be moved to `ACCEPTED`.

---

Abbreviations:
* `ESM` - Ecma262 Modules (ES Modules)
* `NCJS` - Node Modules (a CommonJS variant)

The intent of this standard is to:

* implement interoperability for ESM and Node's existing module system
* create a **Registry Object** (see WHATWG section below) compatible with
  the [WHATWG Loader](http://whatwg.github.io/loader/) Registry and/or [WHATWG module map](https://html.spec.whatwg.org/multipage/webappapis.html#module-map)

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

* **[ModuleNamespaceCreate]
(https://tc39.github.io/ecma262/#sec-modulenamespacecreate)**
    - Provides a means of creating a list of exports manually, used so that
      NCJS `module.exports` can create `ModuleRecord`s that are
      prepopulated.

## 3. Semantics

### 3.1. Determining if source is an ES Module

A new file type will be used for ES Modules, `.mjs`. Files with this extensions
will be treated using the loading semantics in this document, but retain a way
to load existing extensions of `.json`, `.node`, and `.js`. The file type for
JavaScript with IANA as an official file type needs to have this extensions
added. This comes from browsers being unable to implement a new MIME for the
differing parse goals. In order to make this change, contact the IESG.

The `.mjs` file extension will be searched for prior to any `.js` file in any
algorithm that uses searching (`require` or `import`); e.g. once the Node
resolution algorithm reaches file expansion, `path + '.mjs'` would be searched
prior to `path + '.js'` when performing `require(path)`.

# 3.1.1. Inter package loading using file extension breakage.

There is knowledge of breakage for code that upgrades inner package
dependencies such as `require('foo/bar.js')`. As `bar.js` may move to
`bar.mjs`. Since `bar.js` is not the listed entry point this was considered
acceptable.

### 3.2. Import Path Parsing

`import` will parse its path using URLs. As such, encoding and decoding will automatically be performed. This may affect file paths containing any of the following characters: `:`,`?`,`#`, or `%`. Details of the parsing algorithm are at the [WHATWG URL Spec](https://url.spec.whatwg.org/)

* paths with `:` face multiple variations of path mutation
* paths with `%` in their path segments would be decoded
* paths with `?`, or `#` in their paths would face truncation of pathname

### 3.3. Import Path Resolution

The `import` resolution algorithm for node is as follows:

* `request` should represent the requested path to load.
* `context_url` should represent the current script's absolute URL's directory
  * for CLI usage, it is the absolute URL for the current working directory

* if `request` parses via the [URL parser](https://url.spec.whatwg.org/#concept-url-parser)
  * let `url` be the result
* else if `^[.]?[.]?[/]` prefixes `request`
  * let `url` be the result of `new URL(request, context_url)`
  * if `url` points to a directory
    * let `url` be the result of searching the directory
  * if `url` does not point to a file
   * for the well known file extensions `extension` in `[
     .mjs', '.js', '.json', and '.node']`
     * let `searchUrl` be a copy of `url` with `extension` added to the pathname
     * if `searchUrl` points to a file
       * let url be `searchUrl`
       * break
* else
  * let `url` be the result of searching node_modules using `request` and `context_url`
  * NOTE: checks for escaping modules to `node_modules/` via `../` going to be added
* load url
 * we should support `data:` and `file:` out of the box
  * `file:` should use file type to determine dependency type (ES Module / JSON / C++ / NCJS).
  * `data:` should interpret javascript MIME as ES Module. ES Modules share the MIME with NCJS, so we don't have enough data to differentiate, so just assume ES Module since that is [what the browser will assume](https://html.spec.whatwg.org/multipage/webappapis.html#hostresolveimportedmodule(referencingmodule,-specifier)).


`import` will perform non-exact searches on relative or
absolute paths, preserving the behavior in `require()`. This means that known
file extensions, and index files will be searched.

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

#### 3.3.1. Removal of non-local dependencies

All of the following will not be supported by the `import` statement:

* `$NODE_PATH`
* `$HOME/.node_modules`
* `$HOME/.node_libraries`
* `$PREFIX/lib/node`
* `module/../../outside-of-module`

Use local dependencies, and symbolic links as needed.

##### 3.3.1.1. How to support non-local dependencies

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

#### 3.3.2. Errors from new path behavior.

In the case that an `import` is unable to find a module, Node should
make a **best effort** to see if `require` would have found the module and
print out where it was found, if `NODE_PATH` was used, if `HOME` was used, etc.

#### 3.3.3. Shipping both ESM and NCJS

When a `package.json` main is encountered, file extension searches are used to
provide a means to ship both ESM and NCJS variants of packages. If we have two
entry points `index.mjs` and `index.js` setting `"main":"./index"` in
`package.json` will make Node pick up either, depending on what is supported.

##### 3.3.3.1. Excluding main

Since `main` in `package.json` is entirely optional even inside of npm
packages, some people may prefer to exclude main entirely in the case of using
`./index` as that is still in the Node module search algorithm.

### 3.4. ESM Evaluation

#### 3.4.1. Environment Variables

ESM will not be bootstrapped with magic variables and will await upcoming specifications in order to provide such behaviors in a standard way. As such, the following variables are changed:

| Variable | Exists | Value |
| ---- | ---- | --- |
| this | y | [`undefined`](https://tc39.github.io/ecma262/#sec-module-environment-records-getthisbinding) |
| arguments | n | |
| require | n | |
| module | n | |
| exports | n | |
| __filename | n | |
| __dirname | n | |

Like normal scoping rules, if a variable does not exist in a scope, the outer scope is used to find the variable. Since ESM are always strict, errors may be thrown upon trying to use variables that do not exist globally when using ESM.

##### 3.4.1.1. Workaround

Although heavily advised against, you can have a NM module sibling for your ESM that can export these things:

```js
// expose.js
module.exports = {__dirname};
```

```js
// use.mjs
import {__dirname} from './expose.js';
```

#### 3.4.2. Timing

When loading ESM or using any form of `import`, the stack will unwind prior to performing any resolution or evaluation.

##### 3.4.2.1. Example

```javascript
// entry.js
require('one.mjs');
console.log('two');
```

```javascript
// one.mjs
console.log('one');
```

```sh
> node entry.js
two
one
```

### 3.5. Cross Module System Communication

ESM and NCJS differ in many ways, as such they need a well defined means of converting between the two module types and systems.

### 3.5.1. NCJS to ESM

After *any* NCJS finishes evaluation, it will be placed into the same cache as ESM.
The value of what is placed in the cache will reflect a single default export pointing to the value of `module.exports` at the time evaluation ended.

Essentially after any NCJS completes evaluation:

1. if there was an error, place the error in the ESM cache and return
2. let `export` be the value of `module.exports`
3. if there was an error, place the error in the ESM cache and return
4. create an ESM with `{default:export}` as its namespace
5. place the ESM in the ESM cache

### 3.5.2. ESM to NCJS

ESM are *never* placed in the NCJS cache (`require.cache`).

If `require` resolves to an ESM source text, the following steps are taken.

1. let `ret` be a newly created Promise
2. queue a load job on the event loop
  1. if loading completes normally resolve `ret` to the ESM Module Namespace
  2. if an error occurs reject `ret`
3. put a "fetching" entry in the ESM cache
4. return `ret` 

##### 5.4.1.1. Examples

Given:

```javascript
// NCJS.js
module.exports = {
  default:'my-default',
  thing:'stuff'
};
```

You will grab `module.exports` when performing an ES import.

```javascript
// es.mjs

// grabs the namespace
import * as baz from './NCJS.js';
// baz = {
//   default => module.exports;
// }

// grabs "default", aka module.exports directly
import foo from './NCJS.js';
// foo = module.exports;

// grabs "default", aka module.exports directly
import {default as bar} from './NCJS.js';
// bar = module.exports;
```

------

Given:

```javascript
// NCJS.js
module.exports = null;
```

You will grab `module.exports` when performing an ES import.

```javascript
// es.mjs
import foo from './NCJS.js';
// foo = null;

import * as bar from './NCJS.js';
// bar = {default=null};
```

------

Given:

```javascript
// NCJS.js
module.exports = function two() {
  return 2;
};
```

You will grab `module.exports` when performing an ES import.

```javascript
// es.mjs
import foo from './NCJS.js';
foo(); // 2

import * as bar from './NCJS.js';
bar.name; // undefined
bar.default(); // 2
bar(); // throws, bar is not a function
```

------

Given:

```javascript
// NCJS.js
module.exports = Promise.resolve(3);
```

You will grab `module.exports` when performing an ES import.

```javascript
// es.mjs
import foo from './NCJS.js';
foo.then(console.log); // outputs 3

import * as bar from './NCJS.js';
bar.default.then(console.log); // outputs 3
bar.then(console.log); // throws, bar does not have a .then property
```

### 5.5. NCJS consuming ESM

#### 5.5.1. default exports

ESM only export named values. A "default" export is an export that uses
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
// NCJS.js
const esm = require('./es');
// esm ~= Promise => {
//   default => result_from_evaluating_foo;
// }
esm.then(
  es_namespace => {
    console.log(es_namespace.default);
    // {bar='my-default'}
  }
)
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
// NCJS.js
const esm = require('./es');
// esm ~= Promise => {
//   foo=foo;
//   bar=foo;
//   f=f;
//   c=c;
// }
```

### 5.6. Known Gotchas

All of these gotchas relate to opt-in semantics and the fact that NCJS has a
dynamic loader while ESM has a static loader.

No existing code will be affected.

#### 5.6.1. ESM exports are read only

The objects create by an ESM are [ModuleNamespace Objects][5].

These have `[[Set]]` be a no-op and are read only views of the exports of an ESM. Attempting to reassign any named export will not work, but assigning to
the properties of the exports follows normal rules.

### 5.7. NCJS modules allow mutation of imported modules

NCJS modules have allowed mutation on imported modules. When ES modules are
integrating against NCJS systems like Grunt, it may be necessary to mutate a
`module.exports`.

Remember that `module.exports` from NCJS is directly available under `default`
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

Since we need a consistent time to snapshot the `module.exports` of a NCJS
module. We will execute it immediately after evaluation. Code such as:

```javascript
// bad-NCJS.js
module.exports = 123;
setTimeout(_ => module.exports = null);
```

Will not see `module.exports` change to `null`. All ES module `import`s of the
module will always see `{default=123}`.
