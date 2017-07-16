| Title  | Package.json Module Property |
|--------|------------------------------|
| Author | @guybedford                  |
| Status | DRAFT                        |
| Date   | 2017-07-13                   |

## 1. Background

This proposal specifies the `"module"` property in the package.json, building
on the previous work in the
[In Defense of Dot JS](https://github.com/dherman/defense-of-dot-js/blob/master/proposal.md)
proposal (DDJS), as well as many other discussions.

Instead of supporting the additional `"modules"` and `"modules.root"`
properties from that proposal, this proposal aims to adjust the handling of
`"module"` slightly so that it is the only property supported.

A draft specification of the NodeJS module resolution algorithm with this
adjustment is included in section 5. Draft Specification.

## 2. Motivation

There is still uncertainty as to how exactly to distinguish an ES Module from
a CommonJS module. While `.mjs` and `"use module"`
(https://github.com/tc39/proposal-modules-pragma) act as useful indicators,
these properties act as file-specific indicators of the module format. If we
are to keep the `.js` extension without making `"use module"` mandatory, then
there is also a need for a more global indication that a package contains only
ES modules.

Currently all our JS build tools detect modules in slightly different ways.
The `package.json` `module` property has gained good traction as an entry point
mechanism, but there isn't currently clarity on how exactly this property
behaves in the edge cases and for submodule requires (`pkg/x` imports). Since
tools are currently driving the ecosystem conventions, it is worth refining the
exact conventions with an active specification that can gain support, so that
we can continue to converge on the module contract in NodeJS, and do our best
to avoid incompatibilities in future.

## 4. Proposal

Instead of trying to consider a single unified resolver, we break the behaviour
of NodeJS resolution into two separate resolvers:
* The current resolver as in use today, which will continue to be used to
resolve CommonJS modules from CommonJS modules, to ensure absolutely no
breaking edge cases.
* The new ES Modules resolver, that also has the ability to load CommonJS
modules based on a small extension to the existing resolution algorithm.

When using CommonJS `require`, the legacy resolver would be applied, and when
using ES modules, the new ES module resolver algorithm, as along the lines
specified here would be applied.

**The rule proposed here is that the ES module resolver always loads a module
from package with a "module" property as an ES module, and loads a module from
a package without that property as a CommonJS module (unless it is a .mjs file
or "use module" source).**

Under this rule, the simple cases remain the same as the DDJS proposal:

* A package with only a `main` and no `module` property will be loaded as
containing CommonJS modules only.
* A package with only a `module` property and no `main` property will be loaded
as containing ES Modules only.

The difficult case with the DDJS proposal is the transition case of a package
that contains both a `main` and `module` property - selecting which main entry
point and target to use when loading `pkg` or `pkg/x.js`.

For a package that contains both a `main` and a `module` property -
* When the parent module doing the require is an ES Module, the `module` main
will apply, and any module loaded from the package will be loaded as an ES Module.
* When the parent module doing the require is a CommonJS module, the `main`
main will apply, and any module loaded from the package will be loaded as
a CommonJS Module.

In this way, we continue to support the existing ecosystem with backwards
compatibility, while keeping the scope of the specification as simple as possible.

## 4.1 Public API for Mixed CJS and ES Module Packages

A package delivering both CommonJS and ES Modules would then typically
tell its users to just import via `import {x} from 'pkgName'` or
`require('pkgName').x`, with the `module` and `main` properties applying
respectively.

In the case where a package publicly exposes sub-modules, it would need
to document that the CommonJS and ES Module sources are at different paths -
`import {x} from 'pkgName/submodule.js'` and
`import {x} from 'pkgName/cjs/submodule.js'`. Or simply a `.js` and
 `.mjs` variant, this being the author's preference.

## 4.2 Package Boundary Detection

This proposal, like DDJS, requires that we can get the package configuration
given only a module path. This is based on checking the package.json file
through the folder hierarchy:

* For a given module, the package.json file is checked in that folder,
continuing to check parent folders for a package.json if none is found. If we
reach a parent folder of `node_modules`, we stop this search process.
* When no package.json module property is found, NodeJS would default to
loading any module as CommonJS.

These rules are taken into account in the draft specification included below.

## 4.3 Loading Modules without a package.json

If writing a `.js` file without any `package.json` configuration, it remains
possible to opt-in to ES modules by indicating this by either using the `.mjs`
file extension or `"use module"` directive, which always take preference.

## 4.4 Packages Consisting of both CommonJS and ES Modules

For a package that contains both ES modules in a `lib` folder and CommonJS
modules in a `test` folder, if it was desired to be able to load both formats
with the NodeJS ES Module resolver, the approach that could be taken would be
to have two package.json files - one at the base of the package with a
package.json containing a `module` property, and another in the `test` folder
itself, without any `module` property. The `test` folder package.json would
then take precedence for that subfolder, allowing a partial adoption path.

While this approach is by no means elegant, it falls out as a side effect of
the package detection, and provides an adequate workaround for the transition
phase.

## 4.5 Packages without any Main

For packages without any main entry point that expect submodule requires, a
boolean `"module": true` variation could be supported in the package.json so
that `pkg/x`-style imports can still loaded as ES modules.

## 4.6 Caching

For performance the package.json contents are cached for the duration of
execution (including caching the absence of a package.json file), just like
modules get cached in the module registry for the duration of execution. This
caching behaviour is described in the draft specification here.

## 4.7 Enabling wasm

For future support of Web Assembly, this spec also reserves the file extension
`.wasm` as throwing an error when attempting to load modules with this
extension, in order to allow Web Assembly loading to work by default in future.

# 5. Draft Specification

The `RESOLVE` function specified here specifies the ES Module resolver used
only for ES Module specifier resolution, separate to the existing `require()`
resolver.

It is specified here to return a `Module` object, which would effectively be a
wrapper of the
[V8 Module class](https://v8.paulfryzel.com/docs/master/classv8_1_1_module.html).

> **RESOLVE(name: String, parentPath: String): Module**
> 1. Assert _parentPath_ is a valid file system path.
> 1. If _name_ is a NodeJS core module then,
>    1. Return the NodeJS core _Module_ object.
> 1. If _name_ is a valid absolute file system path, or begins with _'./'_,
_'/'_ or '../' then,
>    1. Let _requestPath_ be the path resolution of _name_ to _parentPath_,
with URL percent-decoding applied and any _"\\"_ characters converted into
_"/"_ for posix environments.
>    1. Return the result of _RESOLVE_MODULE_PATH(requestPath)_, propagating
any error on abrupt completion.
> 1. Otherwise, if _name_ parses as a _URL_ then,
>    1. If _name_ is not a valid file system URL then,
>       1. Throw _Invalid Module Name_.
>    1. Let _requestPath_ be the file system path corresponding to the file
URL.
>    1. Return the result of _RESOLVE_MODULE_PATH(requestPath)_, propagating
any error on abrupt completion.
> 1. Otherwise,
>    1. Return the result of _NODE_MODULES_RESOLVE(name)_, propagating any
error on abrupt completion.

> **RESOLVE_MODULE_PATH(requestPath: String): Module**
> 1. Let _{ main, module, packagePath }_ be the destructured object values of
the result of _GET_PACKAGE_CONFIG(requestPath)_, propagating any errors on
abrupt completion.
> 1. Let _loadAsModule_ be equal to _false_.
> 1. If _module_ is equal to _true_ then,
>    1. Set _main_ to _undefined_.
>    1. Set _loadAsModule_ to _true_.
> 1. If _module_ is a string then,
>    1. Set _main_ to _module_.
>    1. Set _loadAsModule_ to _true_.
> 1. If _main_ is not _undefined_ and _packagePath_ is not _undefined_ and is
equal to the path of _requestPath_ (ignoring trailing path separators) then,
>    1. Set _requestPath_ to the path resolution of _main_ to _packagePath_.
> 1. Let _resolvedPath_ be the result of _RESOLVE_FILE(requestPath)_,
propagating any error on abrubt completion.
> 1. If _resolvedPath_ is not _undefined_ then,
>    1. If _resolvedPath_ ends with _".mjs"_ then,
>       1. Return the resolved module at _resolvedPath_, loaded as an
ECMAScript module.
>    1. If _resolvedPath_ ends with _".json"_ then,
>       1. Return the resolved module at _resolvedPath_, loaded as a JSON file.
>    1. If _resolvedPath_ ends with _".node"_ then,
>       1. Return the resolved module at _resolvedPath_, loaded as a NodeJS
binary.
>    1. If _resolvedPath_ ends with _".wasm"_ then,
>       1. Throw _Invalid Module Name_.
>    1. If _loadAsModule_ is set to _true_ then,
>       1. Return the resolved module at _resolvedPath_, loaded as an
ECMAScript module.
>    1. If the module at _resolvedPath_ contains a _"use module"_ directive
then,
>       1. Return the resolved module at _resolvedPath_, loaded as an
ECMAScript module.
>    1. Otherwise,
>       1. Return the resolved module at _resolvedPath_, loaded as a CommonJS
module.
> 1. Throw _Not Found_.

> **GET_PACKAGE_CONFIG(requestPath: String): { main: String, format: String,
packagePath: String }**
> 1. For each parent folder _packagePath_ of _requestPath_ in descending order
of length,
>    1. If there is already a cached package config result for _packagePath_
then,
>       1. If that cached package result is an empty configuration entry then,
>          1. Continue the loop.
>       1. Otherwise,
>          1. Return the cached package config result for this folder.
>    1. If _packagePath_ ends with the segment _"node_modules"_ then,
>       1. Break the loop.
>    1. If _packagePath_ contains a _package.json_ file then,
>       1. Let _json_ be the parsed JSON of the contents of the file at
"${packagePath}/package.json", throwing an error for _Invalid JSON_.
>       1. Let _main_ be the value of _json.main_.
>       1. If _main_ is defined and not a string, throw _Invalid Config_.
>       1. Let _module_ be the value of _json.module_.
>       1. If _module_ is defined and not a string or boolean, throw _Invalid
Config_.
>       1. Let _result_ be the object with keys for the values of _{ main,
module, packagePath }_.
>       1. Set in the package config cache the value for _packagePath_ as
_result_.
>       1. Return _result_.
>    1. Otherwise,
>       1. Set in the package config cache the value for _packagePath_ as an
empty configuration entry.
> 1. Return the empty configuration object _{ main: undefined, module:
undefined, packagePath: undefined }_.

> **RESOLVE_FILE(filePath: String): String**
> 1. If _filePath_ is a file, return the real path of _filePath_.
> 1. If _"${filePath}.mjs"_ is a file, return return the real path of
_"${filePath}.mjs"_.
> 1. If _"${filePath}.js"_ is a file, return return the real path of
_"${filePath}.js"_.
> 1. If _"${filePath}.json"_ is a file, return return the real path of
_"${filePath}.json"_.
> 1. If _"${filePath}.node"_ is a file, return return the real path of
_"${filePath}.node"_.
> 1. If _"${filePath}/index.js"_ is a file, return return the real path of
_"${filePath}/index.js"_.
> 1. If _"${filePath}/index.json"_ is a file, return return the real path of
_"${filePath}/index.json"_.
> 1. If _"${filePath}/index.node"_ is a file, return return the real path of
_"${filePath}/index.node"_.
> 1. Return _undefined_.

> **NODE_MODULES_RESOLVE(name: String, parentPath: String): String**
> 1. For each parent folder _modulesPath_ of _parentPath_ in descending order
of length,
>    1. Let _resolvedModule_ be the result of
_RESOLVE_MODULE_PATH("${modulesPath}/node_modules/${name}/")_, propagating any
errors on abrupt completion.
>    1. If _resolvedModule_ is not _undefined_ then,
>       1. Return _resolvedModule_.
