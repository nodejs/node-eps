| Title  | stream.Readable#readInto()  |
|--------|-----------------------------|
| Author | @indutny                    |
| Status | DRAFT                       |
| Date   | 2016-06-01                  |


## 1. Proposal

It is proposed to add `readInto` method and configuration option to the
`stream.Readable` class.

```js
const stream = new stream.Readable({
  readInto(buf, offset, length, callback) {
  }
});

// Asynchronously read data into `buf.slice(offset, offset + length)`
// Invoke continuation with either error (`err`), or original buffer (`buf`),
// original offset (`off`), and number of bytes read (`bytesRead`)
stream.readInto(buf, offset, length, (err, buf, offset, bytesRead) => {
  // ...
});
```

## 2. Rationale

### 2.1 User Perspective

Currently every `Readable#read()` call will either result in an allocation of
a new `Buffer` instance, or will depend on a previously allocated `Buffer`.
Applications like:

* "stringifiers" of incoming data
* Parsers that do not store any intermediate values, or re-emit them as a
  strings or new `Buffer`s
* _..._

...currently incur an extra performance cost and an extra Garbage Collection
load, that is not necessary for their execution. Introduction of `readInto` will
allow skipping this extra-costs through the reuse of the `Buffer`s in
the applications.

### 2.2 Core Perspective

TLS, HTTP is now performed mostly within the C++ layer, raising the complexity
and making things less observable for the JavaScript user code. `readInto` may
be one of the instruments that will help us to move TLS and HTTP logic
implementation back into JavaScript.

## 3. Description of Operation

### 3.1. Compatibility mode

Most of the existing streams implementations won't be modified to provide a
`_readInto` implementation. Thus we need to provide a default one that will copy
the data from the `ReadableState#buffer` to the supplied `Buffer` argument.

On other hand, implementations that have **only** `_readInto` and do not
implement `_read` at all, should still be working. This can be accomplished
by a shim like this:

```js
// NOTE: Just an example, not a particular implementation that we should stick
//       with.
Readable.prototype._read = function _read(size) {
  const buf = Buffer.alloc(size);
  this._readInto(buf, 0, buf.length, (err, length) => {
    if (err)
      return this.emit('error', err);

    this.push(buf.slice(0, length));
  });
};
```

### 3.2. General

Following scheme applies to a general use case:

```js
// NOTE: Just an example, not a particular implementation that we should stick
//       with.
Readable.prototype.readInto = function readInto(buf, off, len, cb) {
  if (this.reading)
    return this.queue.push(() => { this.readInto(buf, off, len, cb); });

  this.reading = true;
  this._readInto(buf, off, len, (err, size) => {
    this.reading = false;
    const item = this.queue.pop();
    if (item)
      item();

    cb(err, buf, off, size);
  });
}
```
