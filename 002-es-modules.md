| Title  | ES Module Interoperability  |
|--------|-----------------------------|
| Author | @bmeck                      |
| Status | DRAFT                       |
| Date   | 2017-03-01                  |

**NOTE:** `DRAFT` status does not mean ESM will be implemented in Node
core. Instead that this is the standard, should Node core decide to implement
ESM. At which time this draft would be moved to `ACCEPTED`.

---

Abbreviations:
* `ESM` - Ecma262 Modules (ES Modules)
* `CJS` - Node Modules (a CommonJS variant)

The intent of this standard is to:

* implement interoperability for ESM and Node's existing CJS module system

## 1. Purpose

1. Allow a common module syntax for Browser and Server.
2. Allow a common set of context variables for Browser and Server.

## 2. Related

[ECMA262](https://tc39.github.io/ecma262/) discusses the syntax and semantics of
related syntax, and introduces ESM.

[Dynamic Import](https://github.com/tc39/proposal-dynamic-import) introduces
`import()` which will be available in all parsing goals.

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

## 3. Semantics

### 3.1. Async loading

ESM imports will be loaded asynchronously. This matches browser behavior.
This means:

* If a new `import` is queued up, it will never evaluate synchronously.
* Between module evaluation within the same graph there may be other work done.
Order of module evaluation within a module graph will always be preserved.
* Multiple module graphs could be loading at the same time concurrently.

### 3.2. Determining if source is an ES Module

A new file type will be recognised, `.mjs`, for ES modules. This file type will
be registered with IANA as an official file type, see
[TC39 issue](https://github.com/tc39/ecma262/issues/322). There are no known
issues with browsers since they
[do not determine MIME type using file extensions](https://mimesniff.spec.whatwg.org/#interpreting-the-resource-metadata).

The `.mjs` file extension will not be loadable via `require()`. This means that,
once the Node resolution algorithm reaches file expansion, the path for
path + `.mjs` would throw an error. In order to support loading ESM in CJS files
please use `import()`.

#### 3.2.1 MIME of `.mjs` files

The MIME used to identify `.mjs` files should be a [web compatible JavaScript MIME Type](https://html.spec.whatwg.org/multipage/scripting.html#javascript-mime-type).

### 3.3. ES Import Path Resolution


ES `import` statements will perform non-exact searches on relative or
absolute paths, like `require()`. This means that file extensions, and
index files will be searched. However, ESM import specifier resolution will be
done using URLs which match closer to the browser. Unlike browsers, only the
`file:` protocol will be supported until network and security issues can be
researched for other protocols.

With `import` being URL based encoding and decoding will automatically be
performed. This may affect file paths containing any of the following
characters: `:`,`?`,`#`, or `%`. Details of the parsing algorithm are at the
[WHATWG URL Spec](https://url.spec.whatwg.org/).

* paths with `:` face multiple variations of path mutation
* paths with `%` in their path segments would be decoded
* paths with `?`, or `#` in their paths would face truncation of pathname

All behavior differing from the
[`type=module` path resolution algorithm](https://html.spec.whatwg.org/multipage/webappapis.html#resolve-a-module-specifier)
will be places in locations that would throw errors in the browser.

#### 3.3.1. Algorithms

##### 3.3.1.1. HostResolveImportedModule Search

Notes:

* The CLI has a location URL of the process working directory.
* Paths are resolved to realpaths normally *after* all these steps.

1. Apply the URL parser to `specifier`. If the result is not failure,
return the result.
2. If `specifier` does start with the character U+002F SOLIDUS (`/`), the
two-character sequence U+002E FULL STOP, U+002F SOLIDUS (`./`), or the
three-character sequence U+002E FULL STOP, U+002E FULL STOP,
U+002F SOLIDUS (`../`)
    1. Let `specifier` be the result of applying the URL parser to `specifier`
    with importing location's URL as the base URL.
    2. Return the result of applying the path search algorithm to `specifier`.
3. Return the result of applying the module search algorithm to `specifier`.

##### 3.3.1.2. Path Search

1. If it does not throw an error, return the result of applying the file search
algorithm to `specifier`.
2. If it does not throw an error, return the result of applying the directory
search algorithm to `specifier`.
3. Throw an error.

##### 3.3.1.3. File Search

1. If the resource for `specifier` exists, return `specifier`.
2. For each file extension `[".mjs", ".js", ".json", ".node"]`
    1. Let `searchable` be a new URL from `specifier`.
    2. Append the file extension to the pathname of `searchable`.
    3. If the resource for `searchable` exists, return `searchable`.
3. Throw an error.

##### 3.3.1.4. Package Main Search

1. If it does not throw an error, return the result of applying the file search
algorithm to `specifier`.
2. If it does not throw an error, return the result of applying the index
search algorithm to `specifier`.
3. Throw an error.

##### 3.3.1.5. Index Search

1. Let `searchable` be a new URL from `specifier`.
2. If `searchable` does not have a trailing `/` in its pathname append one.
3. Let `searchable` be the result of applying the URL parser to `./index` with
`specifier` as the base URL.
4. If it does not throw an error, return the result of applying the file search
algorithm to `searchable`.
5. Throw an error.

##### 3.3.1.6. Directory Search

1. Let `dir` be a new URL from `specifier`.
2. If `dir` does not have a trailing `/` in its pathname append one.
3. Let `searchable` be the result of applying the URL parser to `./package.json`
with `dir` as the base URL.
4. If the resource for `searchable` exists and it contains a "main" field.
    1. Let `main` be the result of applying the URL parser to the `main` field
    with `dir` as the base URL.
    2. If it does not throw an error, return the result of applying the package
    main search algorithm to `main`.
5. If it does not throw an error, return the result of applying the index
search algorithm to `dir`.
6. Throw an error.

##### 3.3.1.7. Module Search

1. Let `package` be a new URL from the directory containing the importing
location. If `package` is the same as the importing location, throw an error.
2. If `package` does not have a trailing `/` in its pathname append one.
3. Let `searchable` be the result of applying the URL parser to
`./node_modules/${specifier}` with `package` as the base URL.
4. If it does not throw an error, return the result of applying the file search
algorithm to `searchable`.
5. If it does not throw an error, return the result of applying the directory
search algorithm to `searchable`.
6. Let `parent` be the result of applying the URL parser to `../` with
`package` as the base URL.
7. If it does not throw an error, return the result of applying the module
search algorithm to `specifier` with an importing location of `parent`.
8. Throw an error.

##### 3.3.1.7. Examples

```javascript
import 'file:///etc/config/app.json';
```

Parseable with the URL parser. No searching.

```javascript
import './foo';
import './foo?search';
import './foo#hash';
import '../bar';
import '/baz';
```

Applies the URL parser to the specifiers with a base url of the importing
location. Then performs the path search algorithm.

```javascript
import 'baz';
import 'abc/123';
```

Performs the module search algorithm.

### 4. Compatibility

#### 4.3.2 Removal of non-local dependencies

All of the following will not be supported by the `import` statement:

* `$NODE_PATH`
* `$HOME/.node_modules`
* `$HOME/.node_libraries`
* `$PREFIX/lib/node`

Use local dependencies, and symbolic links as needed.

##### 4.3.2.1 How to support non-local dependencies

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

#### 4.3.3. Removal of fallthrough behavior.

Exact algorithm TBD.

#### 4.3.4. Errors from new path behavior.

In the case that an `import` statement is unable to find a module, Node should
make a **best effort** to see if `require` would have found the module and
print out where it was found, if `NODE_PATH` was used, if `HOME` was used, etc.

#### 4.4. Shipping both ESM and CJS

When a `package.json` main is encountered, file extension searches are used to
provide a means to ship both ESM and CJS variants of packages. If we have two
entry points `index.mjs` and `index.js` setting `"main":"./index"` in
`package.json` will make Node pick up either, depending on what is supported.

##### 4.4.1. Excluding main

Since `main` in `package.json` is entirely optional even inside of npm
packages, some people may prefer to exclude main entirely in the case of using
`./index` as that is still in the Node module search algorithm.

### 4.5. ESM Evaluation

#### 4.5.1. Environment Variables

ESM will not be bootstrapped with magic variables and will await upcoming
specifications in order to provide such behaviors in a standard way. As such,
the following variables are changed:

| Variable | Exists | Value |
| ---- | ---- | --- |
| this | y | [`undefined`](https://tc39.github.io/ecma262/#sec-module-environment-records-getthisbinding) |
| arguments | n | |
| require | n | |
| module | n | |
| exports | n | |
| __filename | n | |
| __dirname | n | |

Like normal scoping rules, if a variable does not exist in a scope, the outer
scope is used to find the variable. Since ESM are always strict, errors may be
thrown upon trying to use variables that do not exist globally when using ESM.

##### 4.5.1.1. Standardization Effort

[Efforts are ongoing to reserve a specifier](https://github.com/whatwg/html/issues/1013)
that will be compatible in both Browsers and Node. Tentatively it will be
`js:context` and export a single `{url}` value.

##### 4.5.1.2. Getting CJS Variables Workaround

Although heavily advised against, you can have a CJS module sibling for your
ESM that can export these things:

```js
// expose.js
module.exports = {__dirname};
```

```js
// use.mjs
import expose from './expose.js';
const {__dirname} = expose;
```

### 4.6. ES consuming CommonJS

After *any* CJS finishes evaluation, it will be placed into the same cache as
ESM. The value of what is placed in the cache will reflect a single default
export pointing to the value of `module.exports` at the time evaluation ended.

Essentially after any CJS completes evaluation:

1. if there was an error, place the error in the ESM cache and return
2. let `export` be the value of `module.exports`
3. if there was an error, place the error in the ESM cache and return
4. create an ESM with `{default:module.exports}` as its namespace
5. place the ESM in the ESM cache

Note: step 4 is the only time the value of `module.exports` is assigned to the
ESM.

#### 4.6.1. default imports

`module.exports` is a single value. As such it does not have the dictionary
like properties of ES module exports. In order to transform a CJS module into
ESM a `default` export which will point to the value of `module.exports` that
was snapshotted *imediately after the CJS finished evaluation*. Due to problems
in supporting named imports, they will not be enabled by default. Space is
intentionally left open to allow named properties to be supported through
future explorations.

##### 4.6.1.1. Examples

Given:

```javascript
// cjs.js
module.exports = {
  default:'my-default',
  thing:'stuff'
};
```

You will grab `module.exports` when performing an ESM import of `cjs.js`.

```javascript
// es.mjs

import * as baz from './cjs.js';
// baz = {
//   get default() {return module.exports;},
// }

import foo from './cjs.js';
// foo = module.exports;

import {default as bar} from './cjs.js';
// bar = module.exports
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

You will grab `module.exports` when performing an ESM import.

```javascript
// es.mjs
import foo from './cjs.js';
foo(); // 2

import * as bar from './cjs.js';
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

### 4.7. CommonJS consuming ES

#### 4.7.1. default exports

ES modules only export named values. A "default" export is an export that uses
the property named `default`.

##### 4.7.1.1. Examples

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
const es_namespace = await import('./es');
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
const es_namespace = await import('./es');
// es_namespace ~= {
//   get foo() {return foo;}
//   get bar() {return foo;}
//   get f() {return f;}
//   get c() {return c;}
// }
```

### 4.8. Known Gotchas

All of these gotchas relate to opt-in semantics and the fact that CommonJS is a
dynamic loader while ES is a static loader.

No existing code will be affected.

#### 4.8.1. ES exports are read only

The objects create by an ES module are [ModuleNamespace Objects][5].

These have `[[Set]]` be a no-op and are read only views of the exports of an ES
module. Attempting to reassign any named export will not work, but assigning to
the properties of the exports follows normal rules. This also means that keys
cannot be added.

### 4.9. CJS modules allow mutation of imported modules

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

#### 4.9.1. ES will not honor reassigning `module.exports` after evaluation

Since we need a consistent time to snapshot the `module.exports` of a CJS
module. We will execute it immediately after evaluation. Code such as:

```javascript
// bad-cjs.js
module.exports = 123;
setTimeout(_ => module.exports = null);
```

Will not see `module.exports` change to `null`. All ES module `import`s of the
module will always see `123`.

## 5. JS APIs

* `vm.Module` and ways to create custom ESM implementations such as those in
[jsdom](https://github.com/tmpvar/jsdom).
* `vm.ReflectiveModule` as a means to declare a list of exports and expose a
reflection API to those exports.
* Providing an option to both `vm.Script` and `vm.Module` to intercept
`import()`.

* Loader hooks for:
    * Rewriting the URL of an `import` request *prior* to loader resolution.
    * Way to insert Modules a module's local ESM cache.
    * Way to insert Modules the global ESM cache.