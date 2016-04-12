| Title  | Tile of EP      |
|--------|-----------------|
| Author | @jasnell        |
| Status | DRAFT           |
| Date   | 2016-04-11      |

## Description

NodeSource's n|solid Node.js build introduces the notion of
[policies](https://docs.nodesource.com/nsolid/1.2/docs/policies) that can
be applied to Node to essentially sandbox applications.

Example n|solid policy document:

```json
{
  "// severity levels": "0 - ignore; 1 - warning; 2 - throw error; 3 - exit",

  "blacklist": {
    "modules": {
      "dns": 2
    },
    "bindings": {
      "cares_wrap": 2
    }
  }
}
```

The `blacklist` property identifies specific core modules and bindings that
can or cannot be required. n|solid will either print a warning to the console,
throw an error or abort the process of a blacklisted module is required.

This EP proposes a modified form of this policy mechanism with a slightly 
refactored policy syntax built into Node.js core for v7.

## Syntax

* `0` : Grant
* `1` : Warn
* `2` : Deny (throw)
* `3` : Abort (Exit)

```json
{
  "*": {
    "default": 0,
    "modules": {
      "dns": 2
    },
    "bindings": {
      "cares_wrap": 2
    }
  },
  "module_a": {
    "modules": {
      "dns": 0,
      "fs": 2
    },
    "bindings": {
      "cares_wrap": 0,
      "fs": 2
    }
  }
}
```

In this example, all modules are denied access to DNS except for "module_a".
However, "module_a" is denied access to the FS module and bindings.

The top level keys correspond to modules, with the special value `*` serving
as a wild card for all modules. The keys are expressed either as names or as
paths relative to the location of the Node.js app.

```json
{
  "*": {
    "default": 0,
    "modules": {
      "dns": 2
    },
    "bindings": {
      "cares_wrap": 2
    }
  },
  "./module_a": {
    "modules": {
      "dns": 0,
      "fs": 2
    },
    "bindings": {
      "cares_wrap": 0,
      "fs": 2
    }
  }
}
```

Assuming a file `/Users/sally/test.js`, the `./module_a` policy would apply
to a module required at `/Users/sally/module_a`.

### JSON Schema for Syntax

```JSON
{
  "type": "object",
  "$ref": "#/definitions/policy",
  "definitions": {
    "range": {
      "type": "integer",
      "minimum": 0,
      "maximum": 3
    },
    "policy": {
      "type": "object",
      "properties": {
        "*": { "$ref": "#/definitions/module" }
      },
      "patternProperties": {
        "^.+$": { "$ref": "#/definitions/module" }
      }
    },
    "module": {
      "type": "object",
      "properties": {
        "default": {"$ref": "#/definitions/range"},
        "modules": {
          "$ref": "#/definitions/modules-policy"
        },
        "bindings": {
          "$ref": "#/definitions/bindings-policy"
        }
      }
    },
    "bindings-policy": {
      "type": "object",
      "properties": {
        "default": {"$ref": "#/definitions/range"},
        "async_wrap": {"$ref": "#/definitions/range"},
        "buffer": {"$ref": "#/definitions/range"},
        "cares_wrap": {"$ref": "#/definitions/range"},
        "contextify": {"$ref": "#/definitions/range"},
        "crypto": {"$ref": "#/definitions/range"},
        "fs": {"$ref": "#/definitions/range"},
        "fs_event_wrap": {"$ref": "#/definitions/range"},
        "http_parser": {"$ref": "#/definitions/range"},
        "js_stream": {"$ref": "#/definitions/range"},
        "os": {"$ref": "#/definitions/range"},
        "pipe_wrap": {"$ref": "#/definitions/range"},
        "process_wrap": {"$ref": "#/definitions/range"},
        "signal_wrap": {"$ref": "#/definitions/range"},
        "spawn_sync": {"$ref": "#/definitions/range"},
        "stream_wrap": {"$ref": "#/definitions/range"},
        "tcp_wrap": {"$ref": "#/definitions/range"},
        "timer_wrap": {"$ref": "#/definitions/range"},
        "tls_wrap": {"$ref": "#/definitions/range"},
        "tty_wrap": {"$ref": "#/definitions/range"},
        "udp_wrap": {"$ref": "#/definitions/range"},
        "uv": {"$ref": "#/definitions/range"},
        "v8": {"$ref": "#/definitions/range"},
        "zlib": {"$ref": "#/definitions/range"}
      }
    },
    "modules-policy": {
      "type": "object",
      "properties": {
        "default": {"$ref": "#/definitions/range"},
        "_debug_agent": {"$ref": "#/definitions/range"},
        "_debugger": {"$ref": "#/definitions/range"},
        "_linklist": {"$ref": "#/definitions/range"},
        "assert": {"$ref": "#/definitions/range"},
        "buffer": {"$ref": "#/definitions/range"},
        "child_process": {"$ref": "#/definitions/range"},
        "console": {"$ref": "#/definitions/range"},
        "crypto": {"$ref": "#/definitions/range"},
        "cluster": {"$ref": "#/definitions/range"},
        "dgram": {"$ref": "#/definitions/range"},
        "dns": {"$ref": "#/definitions/range"},
        "domain": {"$ref": "#/definitions/range"},
        "events": {"$ref": "#/definitions/range"},
        "freelist": {"$ref": "#/definitions/range"},
        "fs": {"$ref": "#/definitions/range"},
        "http": {"$ref": "#/definitions/range"},
        "_http_agent": {"$ref": "#/definitions/range"},
        "_http_client": {"$ref": "#/definitions/range"},
        "_http_common": {"$ref": "#/definitions/range"},
        "_http_incoming": {"$ref": "#/definitions/range"},
        "_http_outgoing": {"$ref": "#/definitions/range"},
        "_http_server": {"$ref": "#/definitions/range"},
        "https": {"$ref": "#/definitions/range"},
        "module": {"$ref": "#/definitions/range"},
        "net": {"$ref": "#/definitions/range"},
        "os": {"$ref": "#/definitions/range"},
        "path": {"$ref": "#/definitions/range"},
        "process": {"$ref": "#/definitions/range"},
        "punycode": {"$ref": "#/definitions/range"},
        "querystring": {"$ref": "#/definitions/range"},
        "readline": {"$ref": "#/definitions/range"},
        "repl": {"$ref": "#/definitions/range"},
        "stream": {"$ref": "#/definitions/range"},
        "_stream_readable": {"$ref": "#/definitions/range"},
        "_stream_writable": {"$ref": "#/definitions/range"},
        "_stream_duplex": {"$ref": "#/definitions/range"},
        "_stream_transform": {"$ref": "#/definitions/range"},
        "_stream_passthrough": {"$ref": "#/definitions/range"},
        "_stream_wrap": {"$ref": "#/definitions/range"},
        "string_decoder": {"$ref": "#/definitions/range"},
        "sys": {"$ref": "#/definitions/range"},
        "timers": {"$ref": "#/definitions/range"},
        "tls": {"$ref": "#/definitions/range"},
        "_tls_common": {"$ref": "#/definitions/range"},
        "_tls_legacy": {"$ref": "#/definitions/range"},
        "_tls_wrap": {"$ref": "#/definitions/range"},
        "tty": {"$ref": "#/definitions/range"},
        "url": {"$ref": "#/definitions/range"},
        "util": {"$ref": "#/definitions/range"},
        "v8": {"$ref": "#/definitions/range"},
        "vm": {"$ref": "#/definitions/range"},
        "zlib": {"$ref": "#/definitions/range"}
      }
    }
  }
}
```

## Command Line Flags

A new `--policy` command line flag (or `--policies` as used by n|solid), would
be used to specify the location of the policy JSON file.

## API

A read-only `require.policy` property will return the policy currently in
effect for the current module.

## Cluster and Child Processes

The `--policy` flag would be automatically set on Node.js child processes
launched using cluster or child_process.

## Other deltas from n|solid

The policy document would deal solely with sandboxing require. The n|solid
`zeroFillAllocations` mechanism would not be included and the policy file
should not be extended for any other purpose that sandboxing require.
