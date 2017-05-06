| Title  | Staged require() with lifecycle hooks |
|--------|---------------------------------------|
| Author | @qard                                 |
| Status | DRAFT                                 |
| Date   | 2017-05-05                            |

# Proposal

There are two major purposes to this proposal:
- To provide module loading lifecycle hooks
- To split resolving, loading and compilation concerns

# Motivations

Currently some very popular userland modules depend heavily on monkey-patching
to provide their functionality. This can be unsafe, inflexible and damaging to
performance.

Some examples include:

- Tracing, debugging and monitoring
- Code coverage
- Mocking
- Transpiling

### Safety

Monkey-patching can be unsafe for many reasons.

Sometimes the behaviour of code changes based on the parameter length of a
function given as input to another, such as the `(req, res)`, `(req, res, next)`
and `(err, req, res, next)` route handler signatures in express, which will
switch the first argument to being an error instance if the function parameter
arity is 4. Monkey-patching express route handlers properly requires branching
instrumentation patches which provide different patched functions to match the
length of the original route handler. This is easy to miss or get wrong, which
will generally result in a crashed app.

Sometimes patches can interfere with getter/setter behaviour. An example of this
is `pg.native` mutating the rest of the postgres driver module to expect the
native bindings to be used. Dynamic instrumentation is unaware of what the app
might want to do in the future, so it needs to patch this optimistically, but
if one naively tries to just do `const oldProp = pg.native; pg.native = newProp`
it will trigger the getter behaviour, breaking the JS interface. Again, this is
easy to miss, likely resulting in a crashed app.

It can be difficult to differentiate between async callbacks that need code to
maintain barrier linkage and sync callbacks which do not. Some interfaces also
accept multiple function arguments or callbacks stored in an options object
which could contain other functions, so dynamically identifying callbacks that
need to be instrumented can be unreliable or even impossible.

Any form of queueing, even simply putting a callback in an array to be called
elsewhere, will break continuation linking and requires manual intervention to
maintain that linkage which is not always possible for dynamic instrumentation.

Most APM providers try/catch constantly because the existing solutions for
tracking continuation lose context regularly. We've done the best we can with
monkey-patching, but we've hit a wall and have been pushing for things like
`async-hooks` to improve the situation. However, `async-hooks` only solves part
of the problem, linking the callsite to the callback. However, context on what
was happening still requires monkey-patching, and there are plenty of situations
where context-loss is not the hard part.

At the code text level, applying heuristic behaviour analysis is possible to
more deeply understand the complex interaction of the code, enabling a much
wider range of possibilities in terms of dynamic instrumentation.

### Performance

Monkey-patching is prone to performance issues. The vast majority requires
wrapping things in closures. More complex instrumentation can often trigger
deoptimization. Sometimes heavy runtime checking is required with constant
dynamic function generation and reassignment, which can put a lot of undue
stress on the optimizer and garbage collector.

If one could intercept the module loader and apply code transformations directly
to the code text before compilation, there's a great deal that could be gained.
Most, if not all, run-time checking would simply disappear. Most closures would
disappear. Context loss would mostly disappear because the transformer could
take advantage of lexical scope to describe linkage directly at callsites.

### Flexibility

Many things are just not possible to any reasonable capacity relying purely on
monkey-patching. Often things which could provide valuable coverage or tracing
insight are simply not exposed.

An example of a difficult thing to apply tracing hooks to is the timing from
when a socket first enters JS in `net.js` to when it switches to a TLS socket.
One could listen for the `secureConnection` event, but there are several
failure paths between the `TLSSocket` constructor and the event handler being
triggered, which could result in context loss. One might also think to patch
`TLSSocket.prototype._init`, however that will be called by both servers and
clients, which are difficult to differentiate from in this case. To correctly
identify and correlate a net socket to a corresponding TLS upgrade takes
extensive monkey-patching with a lot of run-time checks in a major hot-path.
With an AST transform, a single line of code could be inserted
[before the `TLSSocket` construction][1] in the `Server` constructor.

### ES Modules

A big issue on the horizon is that, if ES Modules are adopted, modules become
immutable at build time. This means that the current monkey-patching approach
will not work anymore. Members of TC39 have already suggested AST transforms
to deal with this issue.

Another concern with ES Modules is the difference in loader behaviour. This is
where splitting resolution and loading concerns from compilation is valuable.

# Implementation

The idea is to go for something similar to the existing `Module._extensions`
interface, but with three stages: resolving, loading and compiling.

Each stage would interact with a `ModuleRequest` object which expresses the
lifetime of a module request through the `resolve`, `load` and `compile` stages.
Initially, it would only include the `parent` and `inputPath` fields. Other
fields would be populated in the different lifecycle stages.

### `ModuleRequest`

##### `moduleRequest.parent`

The parent module which initiated the `require()` request.

##### `moduleRequest.inputPath`

The input string to the `require()` call or `import` statement which initiated
this module request.

##### `moduleRequest.resolvedPath`

The fully resolved file path, as determined by the `require.resolvers` handler.
This would be added to the module request object during the `resolver` stage.

##### `moduleRequest.contents`

The content string (or buffer?) of the module. This field is populated during
the `loader` stage.

##### `moduleRequest.module`

The final module object, built by the `compiler` stage.

### `require.resolvers`

It is the responsibility of `require.resolvers` extension handlers to receive
the in-process `Module` object and use the metadata on it to figure out how to
resolve a load. For `require()` relative location loading, it'd simply look at
the `parent` and resolve the input string of the `require()` call relative to
the `parent` module.

This would roughly correspond to the current `Module._resolveLookupPaths`
implementation, but it should instead receive the module request object which
it will modify in-place.

```js
const mockedPaths = {
  'foo.js': path.join(__dirname, 'mocked-foo.js')
}

const originalResolver = require.resolvers['require']
require.resolvers['require'] = function customResolver(request) {
  if (Object.keys(mockedPaths).includes(request.inputPath)) {
    return mockedPaths[request.inputPath]
  }
  originalResolver(request)
}
```

By defining named behaviour specific to `require()`, we open the door for
supporting the different behaviour of `import` in the future.

### `require.loaders`

The `require.loaders` extension handlers receive the `Module` object after the
`resolver` stage and use the resolved path to attempt to load the contents of
the module file.

```js
const originalLoader = require.loaders['.js']
require.loaders['.js'] = function customLoader(request) {
  // This populates the `contents` field
  originalLoader(request)

  request.contents = request.contents.replace(
    /module\.exports\s+\=/,
    // Don't forget that `exports` object now!
    'exports = module.exports ='
  )
}
```

### `require.compilers`

The `require.compilers` extension handlers receive the `Module` object after the
`loader` stage and pass the `contents` into a compiler function to build the
final module object.

```js
require.compilers['.yaml'] = function yamlCompiler(request) {
  request.module.exports = yaml.parse(request.contents)
}
```

# Notes

- The `ModuleRequest` object could possibly also just be a `Module` instance,
but the current resolution stage occurs before attempting to build a module
instance, so it's possible backwards-incompatible changes may be required to
do that, thus the safer approach of an entirely new object.

[1]: https://github.com/nodejs/node/blob/master/lib/_tls_wrap.js#L829
