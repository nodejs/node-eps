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

Monkey-patching can be unsafe for many reasons. Sometimes code the behaviour
of code changes based on parameter length of input functions--express is a good
example of that. Sometimes patches can interfere with getter/setter behaviour.
It can be difficult to differentiate between async callbacks that need code to
maintain barrier linkage and sync callbacks which do not. Some interfaces
accept multiple function arguments and dynamically identifying which is a
callback that needs to be instrumented can be unreliable. Any form of queueing,
even simply putting a callback in an array to be called elsewhere, will break
continuation linking and requires manual intervention to maintain that linkage
which is not always possible for dynamic instrumentation.

Most APM providers try/catch constantly because the existing solutions for
tracking continuation lose context regularly. We've done the best we can with
monkey-patching, but we've hit a wall and have been pushing for things like
`async-hooks` to improve the situation. However, `async-hooks` only solves part
of the problem--linking the callsite to the callback--but context on what was
happening still requires monkey-patching, and there are plenty of situations
where context-loss is not the hard part.

### Performance

Monkey-patching is prone to performance issues. The vast majority requires
wrapping things in closures. More complex instrumentation can often trigger
deoptimization. Sometimes heavy runtime checking is required with constant
dynamic function generation and reassignment, which can put a lot of undue
stress on the optimizer and garbage collector.

If, for example, one could intercept the module loader and apply an AST
transform before compilation, there's a great deal that could be gained.
Most Run-time checking would simply disappear. Most closures would disappear.
Context loss would disappear because the transformer could take advantage of
lexical scope to describe linkage directly at callsites.

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
will not work anymore. TC39 has already suggested using AST transformation
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
