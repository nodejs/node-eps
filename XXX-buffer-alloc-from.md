# Buffer.alloc / Buffer.allocUnsafe / Buffer.from

Status | Draft
------ | ----
Author | James M Snell
Date   | 25 Jan 2016

## Purpose

* Soft-Deprecate the existing `Buffer()` constructors and introduce new
  `Buffer.from()`, `Buffer.alloc()` and `Buffer.allocUnsafe()` factory methods
  to address Buffer API usability and security concerns.
* Add `--zero-fill-buffers` command line option to node to force all Buffers
  to zero-fill when created

## Details

The details of the proposal are captured by [Pull Request 4682][] in the
nodejs/node repository (https://github.com/jasnell/node/tree/buffer-safe).

### Buffer.alloc(size[, fill])

Allocates a new initialized buffer of `size` bytes. If `fill` is specified
and is a string, the Buffer is allocated off the shared pool by calling:

```
Buffer.allocUnsafe(size).fill(fill);
```

Otherwise, the buffer is initialized by allocating new zero-filled memory

### Buffer.allocUnsafe(size)

Allocates a new uninitialized buffer of `size` bytes off the shared pool. This
is equivalent to calling `new Buffer(size)`

### Buffer.from(value[, encoding])

Replaces the existing `new Buffer(value[, encoding])` constructors.

### `--zero-fill-buffers` command line option

Forces all Buffers to be zero-filled upon allocation.

## Related Discussion

* https://github.com/nodejs/node/issues/4660
* https://github.com/ChALkeR/notes/blob/master/Lets-fix-Buffer-API.md

[Pull Request 4682]: https://github.com/nodejs/node/pull/4682
