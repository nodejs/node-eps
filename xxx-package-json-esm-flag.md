| Title  | Package.json Module Property |
|--------|------------------------------|
| Author | @guybedford                  |
| Status | DRAFT                        |
| Date   | 2017-07-13                   |

## 1. Background

This proposal specifies an `"esm"` flag in the package.json, building
on the previous work in the interop space including
[In Defense of Dot JS](https://github.com/dherman/defense-of-dot-js/blob/master/proposal.md)
proposal (DDJS), as well as many other discussions.

Instead of supporting a `"module"` property or other mechanism,
this proposal simply suggests an `"esm": true` flag in a package.json.

A draft specification of the NodeJS module resolution algorithm with this
adjustment is included in section 5. Draft Specification.

## 2. Motivation

There is still uncertainty as to how exactly to distinguish an ES Module from
a CommonJS module. While `.mjs` can act as a useful indicator, it is still a 
file-specific indicator of the module format. If we are to keep the `.js`
extension without making adding metadata to sources, then there is also a 
need for a more global indication that a package contains only ES modules.

From a user perspective, there are many possible reasons for wanting to
continue to use the `.js` extension. The standard way of loading modules in the
browser for JS users in future will likely be by creating a file, and loading
it with `<script type="module" src="file.js">`. There would be confusion if not
everyone gets on board with the `.mjs` file extension for every workflow
including browser workflows like this, as users wouldn't necessarily understand
why they need to change their code to run correctly in NodeJS when moving
between environments. In addition, using a `.js` extension is the way that all
users deal with ES modules already through build tools, so there will be some
friction trying to alter these existing habits. Regardless of the underlying
reasons, as long as there are a group of users vocally attached to this
extension, their opinions should at least be considered, and that is the aim
of this proposal to provide a path for the `.js` file extension to be used
for future JS modules.

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
from package with an `"esm": true` property in its package.json file as an ES
module, and loads a module from a package without that property as a CommonJS
module (unless it is a .mjs file).**

This is simpler than the `"module"` property in the package.json because it
does not interfere in making the main entry point detection any different to
how it already works today, it only changes the parsing default.

## 4.1 Package Boundary Detection

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
possible to opt-in to ES modules by indicating this by using the `.mjs`
extension.

When running NodeJS code via `node -e` or similar, an alternate flag could
possibly be considered to treat the passed source as an ES module.

## 4.4 Packages Consisting of both CommonJS and ES Modules

For a package that contains both ES modules in a `lib` folder and CommonJS
modules in a `test` folder, if it was desired to be able to load both formats
with the NodeJS ES Module resolver, the approach that could be taken would be
to have two package.json files - one at the base of the package with a
package.json containing a `"esm": true` property, and another in the `test` folder
itself, without any `"esm"` property. The `test` folder package.json would
then take precedence for that subfolder, allowing a partial adoption path.

This provides an adequate workaround for the transition phase.

## 4.5 Caching

For performance the package.json contents are cached for the duration of
execution (including caching the absence of a package.json file), just like
modules get cached in the module registry for the duration of execution. This
caching behaviour is described in the draft specification here.

## 4.6 Further module formats in future

This is only needed for ES modules and not for other module formats because
new additions to the runtime such as WebAssembly or binary ast files
would be detected through their extensions (`.wasm` or `.bast` say).

This problem of not knowing how to interpret a `.js` file is unique to the
upgrade path of moving towards ES modules by default. Hence an `"esm"`
package.json property specifically for this purpose is in line with precedent
for an upgrade path of this nature.

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
> 1. Let _{ main, esm, packagePath }_ be the destructured object values of
the result of _GET_PACKAGE_CONFIG(requestPath)_, propagating any errors on
abrupt completion.
> 1. If _esm_ is not defined then,
>    1. Set _esm_ to _false_.
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
>    1. If _esm_ is set to _true_ then,
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
>       1. Let _esm_ be the value of _json.esm_.
>       1. If _esm_ is defined and not a  boolean, throw _Invalid
Config_.
>       1. Let _result_ be the object with keys for the values of _{ main,
esm, packagePath }_.
>       1. Set in the package config cache the value for _packagePath_ as
_result_.
>       1. Return _result_.
>    1. Otherwise,
>       1. Set in the package config cache the value for _packagePath_ as an
empty configuration entry.
> 1. Return the empty configuration object _{ main: undefined, esm: undefined,
packagePath: undefined }_.

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
