| Title  | `node inspect` CLI debugger |
|--------|-----------------------------|
| Author | @jkrems                     |
| Status | DRAFT                       |
| Date   | 2016-09-04                  |

## Description

For the old V8 debugger protocol,
node offers two ways to use it:

1. `node --debug <file>`: Start `file` with remote debugging enabled.
2. `node debug <file>`: Start an interactive CLI debugger for `<file>`.

But for the Chrome inspector protol,
there's only `--inspect` which is roughly equivalent to the `--debug` flag.
It's impossible to use the new protocol without an external tool.

This proposes the addition of the following commands to node:

```
node inspect script.js      # Start interactive debug session for script.js
node inspect <host>:<port>  # Connect to inspector protocol on host:port
```

`node inspect -p <pid>` would be a lot harder to implement than the others.
So while it might be very useful long-term,
this proposal explicitly does not include that variant.

The debug repl should offer the same methods & similar output to the
[current debugger](https://nodejs.org/api/debugger.html).

## Alternatives

* Offer a blessed, stand-alone `node-inspect` tool installed via npm.
* The methods & output could follow `gdb`'s example.
* The methods & output could follow `lldb`'s example.
