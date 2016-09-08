| Title  | ESM Interoperability |
|--------|-----------------------------|
| Author | @bmeck / @fishrock123       |
| Status | DRAFT                       |
| Date   | 2016-08-26                  |

**NOTE:** `DRAFT` status does not mean ESM / NM2 will be implemented in Node
core. Instead that this is the standard, should Node core decide to implement
ESM / NM2. At which time this draft would be moved to `ACCEPTED`.

---

Abbreviations:
* `ESM` - Ecma262 Modules (ES Modules)
* `NCJS` - Node Modules (a CommonJS variant)
* `NM2` - Node v2.0 Modules (Node Modules using the Module parse goal)
* `RMR` - Reflective Module Record (`ModuleRecord` type)

The intent of this standard is to:

* implement interoperability for the Module Parse Goal and Node's existing
module system

## 1. Purpose

Allow Node.js modules to use "Module Mode" **without**:

1. Hitting the Reflective Module Record.
  - The Reflective Module Record is described below as the `ModuleRecord` type.
2. Using Async Module Resolution.
3. Maintaining [Lifetime Module Idempotency](https://tc39.github.io/ecma262/#sec-moduledeclarationinstantiation).

### 1.1. Avoidance of the Reflective Module Record

Note: The RMR is only a collection of direct pointers in a JIT'd environment.

The RMR poses several interoperability and tooling problems:

1. Importing from NCJS has no support for Named Imports.
  - All core modules must stay as NCJS for the forseeable future due to interoperability constraints.
  - As such, all core modules can only have a default export under the RMR.
  - e.g. no `import { readfile } from 'fs';`
2. Inspecting the RMR is impossible
  - Hooks to inspect only pointers is not possible.
  - This renders APMs, tracing tools, test mocking tools, etc, impossible.
3. Wrapping the RMR is impossible.
  - Wrapping only pointers is not possible.
  - Most tooling vendors use this approach in current NCJS due to a lack of hooks.
  - This, however, is not an option under the RMR.
4. Conditional imports & exports are impossible.
  - (The RMR must be constructed at parse time.)
  - This renders writing Native Modules in ESM impossible.
5. Avoids Sync Module Loading recursion problems.
  - Stated in [15.2.1.17 Runtime Semantics: HostResolveImportedModule](https://tc39.github.io/ecma262/#sec-hostresolveimportedmodule)

### 1.2. Avoidance of Async Module Resolution

1. Allows modules to both use Module Mode and be `require()`'d as usual.
  - Prevents any further interop issues related to timing.
  - Allows existing modules to transition to NM2 while dependants can continue using NCJS to require them without any changes.

Example of the unreasonable work required for dependants to transition to full
async loading:
(Required for `await` to be safe.)

Before (synchronously):
```js
const m = require('module')
// do stuff with m...
exports.thing = m.something
```

After (asynchronously):
```js
// Must be done throughout the entirety of any downstream (user) NCJS dep tree
exports = async function() {
  const m = await require('module')
  // do stuff with m...
  return { thing: m.something }
}
```

### 1.3. Avoidance of Lifetime Module Idempotency

1. Allows REPLs to reload local code that may have been edited since a failure.
  - This is also solvable (in a much less user-friendly way) by allowing access the module cache.

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

A new file type will be used for Node v2.0 Modules, `.mjs`. Files with this extension
will be treated using the loading semantics in this document, but retain a way
to load existing extensions of `.json`, `.node`, and `.js`. The file type for
JavaScript with IANA as an official file type needs to have this extensions
added. This comes from browsers being unable to implement a new MIME for the
differing parse goals. In order to make this change, contact the IESG.

The `.mjs` file extension will be searched for prior to any `.js` file in any
algorithm that uses searching (`require`); e.g. once the Node
resolution algorithm reaches file expansion, `path + '.mjs'` would be searched
prior to `path + '.js'` when performing `require(path)`.

# 3.1.1. Inter package loading using file extension breakage.

There is knowledge of breakage for code that upgrades inner package
dependencies such as `require('foo/bar.js')`. As `bar.js` may move to
`bar.mjs`. Since `bar.js` is not the listed entry point this was considered
acceptable.

### 3.2. Importing Modules in NM2

Node v2.0 Modules do not expose `import` and `export`.
Instead, modules are "imported" as usual and as identically as possible as NCJS,
via `require()`.

### 3.2.1 Function Wrap Approach

To achieve this, modules would be wrapped in a function like a regular script
file, but evaluated as a Module under the Module Parse Goal. However, that
wrapper _may_ be exported with an `export` statement under the hood. As such,
it _may_ look like the following:

```js
export default function (exports, require, module, __filename, __dirname) {
  // user code inserted here
}
```

_If_ exporting under the hood is required, the resulting (hidden) promise
would likely be resolved synchronously on `require()`, given we are in
complete control and can guarantee its safety.

### 3.4. NM2 Evaluation

#### 3.4.1. Environment Variables

NM2 will be bootstrapped with standard NCJS "magic" variables.

| Variable | Exists | Value |
| ---- | ---- | --- |
| this | y | ? |
| arguments | y | same as regular scripts |
| require | y | same as regular scripts |
| module | y | same as regular scripts |
| exports | y | same as regular scripts |
| __filename | y | same as regular scripts |
| __dirname | y | same as regular scripts |

Like normal scoping rules, if a variable does not exist in a scope, the outer scope is used to find the variable. Since NM2 are always strict, errors may be thrown upon trying to use variables that do not exist globally when using NM2.

### 4. Caveats

This approach saves existing operability and tooling functionality in exchange
for a couple caveats, namely:

1. `import` & `export` are unimplemented / unavailable to users
2. Top-level scope is unavailable.
3. Top-level async operations are **unsafe**.
  - Runs into complex preemption and pseudo threading problems.
  - As such, top-level `await` is unavailable.

### 5. As a Transitional stage

It is possible that this _could_ be a transitional approach _if_ ESM improves
to having good enough support for tooling and interop that it is favorable
to move to it fully.

Two options may be available in that case to detect an actual ESM:

1. Another file extension (such as `.esm`).
2. Detection of top-level `import` / `export` / `await` statements.
  - Similar or identical to the Unambiguous JavaScript Grammar approach
