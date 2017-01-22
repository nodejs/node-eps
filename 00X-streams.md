| Title  | Invert dependency of readable-stream |
|--------|--------------------------------------|
| Author | @mcollina                            |
| Status | DRAFT                                |
| Date   | 2017-01-26 12:00:00                  |

## Description

The `readable-stream` module is one of the most popular modules on NPM,
with 33 million downloads per month. `readable-stream` is the basis of
most of the ecosystem utilities for manipulating streams, as it provides
a consistent API across all the different versions of Node.js and the
browser: from Node.js v0.8, and IE 9 and Safari 5 as the oldest browsers.

`readable-stream` was introduced to help module authors build the
streams ecosystem across the different versions of Node.js, backporting
v0.10 streams to v0.8.
The code for the `'stream'` module in core *depends on the version of
core* that is being used, and because streams users inherits from
classes like `Readable`, `Writable` or `Transform` a tiny difference
might make a program work on a version of Node but not in another.
Module authors appreciate the stability `readable-stream` provides,
and it is currently in the dependency tree of gulp, mysql, mongodb, webpack
and many other highly-popular libraries.
A good article for what `readable-stream` provides can be found at
https://r.va.gg/2014/06/why-i-dont-use-nodes-core-stream-module.html.

`readable-stream` is an export of the latest stream API from the Node.js,
done via a mixture of babel transpiling and regular expressions. The export
means that `readable-stream` has to follow the Node.js release cycle, and it
is not versioned independently. Specifically, versioning of `readable-stream`
is extremely hard, as *we are currently not bumping major*
to avoid breaking many packages and maintaining multiple lines. The current
situation is not ideal, as `readable-stream` passively follows what is merged
in Node.js.

At the moment `readable-stream` does not have its own documentation, it is the
documentation of whichever version of Node.js is built from. This creates confusion,
as the latest `stream` from Node.js supports some features that cannot be ported
to all versions of Node.js.

This EP proposes to invert that dependency, and have Node.js include
readable-stream from the `deps` folder.

## SEMVER for streams

As things stands right now, `readable-stream` support Node.js from v0.8
upwards, and fairly ancient browsers versions as well. Because of this,
we are currently using babel (and a bunch of regexps) before release,
to support ES5-based vendors. Dropping support for any of those is a
semver-major change, and given the massive amount of dependencies, it
might have a semver-major ripple effect throughout the ecosystem.
In order to avoid a lot of work for maintainers, we aim to minimize those.
At the time of this writing, we plan __not to bump__ semver-major in 2017.

We recognize that the ecosystem is changing at a fast pace, and at some
point we would have to bump the major. If we decide to do that, we would
have to manage two release lines (or more), and backport changes from
Node.js streams to **both**. As things are now, this is highly
impractical to do, and it would be much easier to manage if `readable-streams`
could manage its own versioning.

## Process / Timeline

1. move the code of streams into `readable-stream`
2. amend the build script for the NPM/ecosystem build
3. rewrite the tests to easily check against both the node
   source folder and old versions of Node.js and the browser.
4. port the Node streams documentation to `readable-stream`; a _copy_
   of the docs will be maintained in Node.js.
4. merge back all changes that happened in the meanwhile, and sync up
5. add `nodejs/readable-stream` inside deps of Node.js, and remove the streams
   implementation; to avoid any possible disruption, this will be a
   _semver-major_ change for Node.js.
6. bump *minor* version of `readable-stream`.

## Challenges

1. export git history from node.js
2. handling issues between the two repositories

These challenges need to be adressed with the rest of the node collaborators.

## Role of the Streams WG

The Streams Working Group will still be responsible for maintaining
`readable-stream` and `'streams'` in core, and this EP does not change
anything. However, the role of `readable-stream` is to
`'port stream to older node versions and browsers'`, while after the
inversion of dependency, it would change to `'support Node.js streams in
throughout the different versions of Node and browsers'`.
As it is right now, the Streams WG commits in maintaining the
performance of streams within Node.js.

## Target version of Node

We think we should target this to be shipped in Node 8.
