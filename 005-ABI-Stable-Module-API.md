| Title  | ABI Stable Module API                 |
|--------|---------------------------------------|
| Author | @mhdawson, @stefanbu, @Ianwjhalliday  |
| Status | DRAFT                                 |
| Date   | 2016-12-12                            |

# Introduction

The module ecosystem is one of the features making Node.js the fastest
growing runtime.  An important subset of modules are those which have a
native component. In specific cases it is important to be able to leverage native
code in order to be able to:

* use existing third party components written in C/C++
* access physical devices, for example a serial port
* expose functionality from the operating system not otherwise available
* achieve higher speed than possible in native JavaScript

There is an existing interface (addons) that allows you to build these modules
as a dynamically-linked shared module that can be loaded into Node.js and
used as an ordinary Node.js module (for more details see
https://nodejs.org/api/addons.html)

Unfortunately in the existing implementation the V8
(the JavaScript runtime for Node.js) APIs are exposed to modules. In
addition Google changes these APIs on a regular basis. This results in
the following issues:

* Native modules must be recompiled for each version of Node.js
* The code within modules may need to be modified for a new version of
  Node.js
* It's not clear which parts of the V8 API the Node.js community believes
  are safe/unsafe to use in terms of long term support for that use.
* Modules written against the V8 APIs may or may not be able to work
  with alternate JavaScript engines when/if Node.js supports them.

The Native Abstractions for Node (NAN)
project (https://github.com/nodejs/nan) can be used to limit
the cases when modules must be updated in order to work with new versions
of Node.js but does not address the other issues as it does not
encapsulate the V8 argument types such that modules still need
to be recompiled for new Node.js versions.

# Proposed Action

The goal of this eps is to define an API that:

* Provides a stable API which is not tied to the current
  or further APIs provided by V8
* Allows native modules built as shared libraries to be used
  with different versions of Node.js
* Makes it clear which APIs can use with the expectation
  of not being affected by changes in the JavaScript engine
  or Node.js release
* The API provided by the Node.js runtime will be in C. In
  addition a C++ wrapper which only contains inline functions
  and uses the C APIs so that the changes in the C++ wrapper
  would not require recompilation.

# Background

There have been ongoing discussions on a new module API in both the
community api working group (https://github.com/nodejs/api), other
issues in the Node.js  issue tracker, and the recent community Node.js
face-to-face VM summit (https://github.com/jasnell/vm-summit).

In that summit there was agreement that we wanted to define this
module API.  The general approach was to be:

* Start with NAN, trim down to minimal subset based on module usage
  and inspection.
* Add in VM agnostic methods for object lifecycle management.
* Add in VM agnostic exception management
* Define C/C++ types for the API which are independent from V8
  (NAN currently uses the V8 types).
* Use the NAN examples to work through the API and build up the
  definition of the key methods.
* Validate on most used modules in the ecosystem
* Complete performance analysis to confirm accepted level of overhead.

Notes/details on other related work and the current experimental progress
on implementing this eps can be found in:
https://docs.google.com/document/d/1bX9SZKufbfhr7ncXL9fFev2LrljymtFZdWpwmv7mB9g/edit#heading=h.ln9aj9qjo969

# API definition

**WORK IN PROGRESS, DRIVEN BY MODULES PORTED SO FAR**

This API will be built up from a minimal set. Based on this set
we have been able to port the NaN example and level down.  Additions
will be made as we port additional modules.

There has been enough progress that instead of listing the API methods
here we will point to the header files in the working PoC repos so it
remains up to date:

Core methods:
* [node_jsvmapi_types.h](https://github.com/nodejs/abi-stable-node/blob/api-prototype-6.2.0/src/node_jsvmapi_types.h)
* [node_jsvmapi.h](https://github.com/nodejs/abi-stable-node/blob/api-prototype-6.2.0/src/node_jsvmapi.h)

AsyncWorker support methods:

* [node_asyncapi_types.h](https://github.com/nodejs/abi-stable-node/blob/api-prototype-6.2.0/src/node_asyncapi_types.h)
* [node_asyncapi.h](https://github.com/nodejs/abi-stable-node/blob/api-prototype-6.2.0/src/node_asyncapi.h)


# C++ Wrapper API

The current view is that the API exported by the Node.js binary
must be in C in order to achieve ABI stability. This [document](http://www.oracle.com/technetwork/articles/servers-storage-dev/stablecplusplusabi-333927.html)
outlines some of the challenges with respect to C++ and
ABI stability.

However, we also believe that a C++ wrapper is possible and
would provide ease of use. The key element is that the C++ part must all
be inlined and only use the external functions/types exported
by the C API.

The current set of helpers to provide ease of use are defined in:

[node_api_helpers.h](https://github.com/nodejs/abi-stable-node/blob/api-prototype-6.2.0/src/node_api_helpers.h)

Some of the more complex parts like AsyncWorker will quite likely end up outside
the scope of the ABI stable API itself, and be in a separate component.
For example, they may end up part of a version of NaN that supports the new API.
