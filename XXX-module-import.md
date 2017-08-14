| Title  | module.import(path):Promise |
|--------|-----------------------------|
| Author | @WebReflection              |
| Status | DRAFT                       |
| Date   | 2017-01-23                  |

# Asynchronous `module.import(path)`
There is [a standard effort](https://github.com/tc39/proposal-dynamic-import)
to bring asynchronous module loading to JavaScript.

Such effort will take some time before it will be fully agreed,
and before it will be widely available across all JS engines.

In the meanwhile, the community could benefit from this _opt-in_
proposal, which is 100% backward compatible, as well as future friendly,
being based entirely on what's been agreed already by TC39.

## The pseudo implementation example
The following code, which works already,
represents how essential is this additional
`module.import` method proposal:
```js
// one added method to Module.prototype
module.constructor.prototype.import =
  function (path) {
    return Promise.resolve()
           .then(() => this.require(path));
  };
```

### Fully backward compatible
Every existent NodeJS/CommonJS module can be imported already.
There is no need to change anything to the existing code-base.
```js
// even core modules work already
module.import('fs').then(fs => {
  // do something with fs module
  fs.readdir('.', console.log);
});
```

### Multiple imports too
Keeping the implementation as simple as possible in order
to have zero conflicts with current dynamic
standard `import` proposal, we can use standard
`Promise.all` method to retrieve at once every needed module.
```js
Promise.all([
  module.import('fs'),
  module.import('path')
]).then(([fs, path]) => {
  fs.readdir(
    path.join(__dirname, 'test.js'),
    console.log
  );
});
```

It's eventually possible to shortcut multiple imports
using the following pattern.
However, it won't be fully compatible
with current standard `import` proposal as it is today.
```js
Promise.all([
  'fs',
  'path'
].map(
  m => module.import(m)
)).then(modules => {});
```

## Unleashing asynchronous exports
While good old modules will keep being usable,
new modules might define themselves as asynchronous
by exporting a `Promise` rather than directly the module.
```js
// example: dir.js
module.exports = Promise.all([
  module.import('fs'),
  module.import('path')
]).then(([fs, path]) => {
  // once all dependencies have been imported
  // return the module that should be imported
  return {
    list(dir) {
      return new Promise((res, rej) => {
        fs.readdir(
          path.resolve(dir),
          (err, list) => {
            if (err) rej(err);
            else res(list);
          }
        );
      });
    }
  };
});

// import example
module.import('./dir').then(dir => {
  dir.list('.').then(console.log);
});
```

## Asynchronous exports: why?
The example with core modules is mostly for
copy and paste testability sake only,
the truth is that many modules have
asynchronous initialization behind databases connections,
remote API calls, and other non blocking operations.

New modules created with async `import(...)` in mind
can early reject when needed, instead of emitting
or throwing unhandled errors in the wild at distance.

Importing a list of 20 modules at once, would be
easy to _catch_ all at once and react accordingly,
as opposite of handling every single module possible
non blocking initialization a part.

On top of that, on the client side,
every module would most likely be preferred as asynchronous,
so that this proposal can improve Universal-JS-ability even more.

## Why not as global function ?
The `import` word is reserved.
Unless the community wants to wait until the official standard
is finalized and the v8 change shipped,
using a `global.import(path)` does not seem even
aligned with the concept of module, which should never even use
`global` reference if not to quickly feature test something,
and does not seem to be universally friendly,
since `global` is not standard outside NodeJS world.

## Why polluting `module` and not `require` ?
There are at least two reasons:

  1. the Web has a role too
  2. historical confusion and semantics

The current proposal is aligned with what
`<script type="module">` should also ship on browsers.
This means that the word `module` is already part
of both ECMAScript and Web specifications.

The word `require`, and the presence of an extra callback
that is strictly coupled with synchronous CommonJS only,
something not considered by dynamic `import` at all
and actually warned in console when similarly operated
on the main thread of any Web page, would only create confusion
in terms of what `module` and `import` means for the present
and the future of the language, in a platform independent way.

On top of that, `require` function has been historically
[abused](https://www.npmjs.com/package/require.async) or
[polluted](http://requirejs.org/docs/api.html#data-main)
in an era where `Promise`, as well as `async` and `await`
patterns, weren't even discussed in the JS community.

Accordingly, adding yet another method to a function
which name and semantics are unknown for the rest of
the JS ecosystem, does not look like a good choice
in terms of new, future friendly, module feature.

On the other hand, being `module` where developers
define their exports, I don't see any confusion
in being `module` where developers can import
their dependencies.

After all, they are going to "_import module_"
or "_export module_", and not going to
"_require an import of a module_" since they never
"_required an export of a module_".

## Why as core feature ? Why not transpilers ?
Transpilers are a great resource to have new
language features implemented in what's there already.

Unfortunately these can also have bugs, like it's been
for [classes](https://github.com/babel/babel/issues/4480)
since ever, so that transpilers become an issue, a stopper,
or a feature that cannot be fully trusted instead.

Using transpilers since ES2015 had various side effects
due un-polyfillable new features such `Symbol`, `WeakMap`,
`class`, `super()`, and shorthand methods without prototype.

Accordingly, requiring transpilers as mandatory tool
to import asynchronously a module, is not necessarily
the most secure, reliable, or plausible advice.

Since this proposal has been tested to be reliable
down to NodeJS 0.12, the first one that shipped with
native `Promise`, I don't see why we should lose
this opportunity to make every CommonJS user
capable right away to export, and import,
asynchronous modules that are future friendly.

## What about proposing this as standard ?
I have already mentioned these points in one open issue [1].

In the very same thread, there are other developers believing
`module`, which is luckily not a reserved word, is the best
object to host `import` capabilities and per each module,
like it is already in CommonJS and NodeJS.

## How can this work on the Web ?
I have already created a tiny ~1KB utility that brings
everything discussed in here to browsers too (all of them) [2].

Despite my utility usage, if the proposal will end up agreeing
on using a `module.import(path)` method, instead of `import(path)`,
there'll still be a `module` object to eventually attach an export,
meaning the future code won't break even for early adopters.

## How can old NodeJS version import new modules ?
They can add the top level implementation in their main entry point,
as polyfill for all required modules, or they can simply
install new modules and require them as such:
```js
var asyncModule = Promise.resolve(
  require('new-module')
).then(function (newModule) {
  // do something
});
```
Following the same approach, NodeJS 0.12 env can also already
create and export asynchronous module.

## When `import` will be official, what happens ?
The same that happened until now with `require`,
the previously mentioned transpilers will have a core
utility to fallback the new syntax that cannot possibly be
natively polyfilled in any JS environment as it is now.



[1] https://github.com/tc39/proposal-dynamic-import/issues/35#issuecomment-274561912

[2] https://github.com/WebReflection/common-js

