| Title  | ICU Module                  |
|--------|-----------------------------|
| Author | @jasnell                    |
| Status | DRAFT                       |
| Date   | 2016-08-11                  |

## Description

The ICU4C library that we use for internationalization contains a significant
array of additional functionality not currently exposed by the EcmaScript 402
standard. Some of this additional functionality would be useful to expose via
a new `'unicode'` module.

## Interface

Initially, the `'unicode'` module would provide methods for the following:

1. One-Shot and Streaming Buffer re-encoding. ICU includes code for converting
   from one encoding to another. This is similar to what is provided by `iconv`
   but it is built in to ICU4C. The `'unicode'` module would include converters
   for *only* the character encodings directly supported by core. Developers
   would continue to use `iconv` or `iconv-lite` (or similar) for more exotic
   things.

```js
const unicode = require('unicode');

// One-shot conversion. Converts the entire Buffer in one go.
// Assumes that the Buffer is properly aligned on UFT-8 boundaries
const myBuffer = Buffer.from(getUtf8DataSomehow(), 'utf8');
const newBuffer = unicode.transcode(myBuffer, 'utf8', 'ucs2');

// Streaming conversion
const transcodeStream = icu.createTranscoder('utf8', 'ucs2');
transcodeStream.on('data', (chunk) => {
  // chunk is a UTF-16 (ucs2) encoded buffer
});
// Writing UTF-8 data
transcodeStream.write(getUtf8DataSomehow());
```

Again, this would ONLY support the encodings for which we already have built-in
support in core (acsii, iso8859-1, utf8 and utf16le).

2. UTF-8 and UTF-16 aware `codePointAt()` and `charAt()` methods for `Buffer`.
   This one is pretty straightforward. They would return either the Unicode
   codepoint or the character at the given byte offset even if the byte offset
   is not on a UTF-8 or UTF-16 lead byte. These are intended to be symmetrical
   with `String.prototype.codePointAt()` and `String.prototype.charAt()`

```js
const unicode = require('unicode');

const myBuffer = Buffer.from('a€bc', 'utf8');

console.log(unicode.codePointAt(myBuffer, 1, 'utf8'));

console.log(unicode.charAt(myBuffer, 1, 'utf8'));
```

3. UTF-16 and UTF-8 aware `slice()` for `Buffer`. This is similar to the
   existing `Buffer.prototype.slice()` except that, rather than byte offsets,
   the `start` and `end` are codepoint/character offsets This would make it
   symmetrical with `String.prototype.slice()` but for Buffers. The advantage
   is that this allows the Buffer to be sliced in a way that ensures proper
   alignment with UTF-8 or UTF-16 encodings.

```js
const unicode = require('unicode');

const myBuffer = Buffer.from('a€bc', 'utf8');

unicode.slice(myBuffer, 'utf8', 1, 3); // returns a Buffer with the utf8
                                       // encoding of €b
```

*Passing in either `ascii` or `binary` would fallback to the current
`Buffer.slice()` behavior.*

## Proof-of-concept

A proof-on-concept implementation that includes everything except the streaming
converters can be found at: 

  https://github.com/jasnell/node/tree/icu-module

The module requires turning on the basic ICU converter switch which has a
nominal impact on binary size but nothing significant. Since the conversions
we would require in core are purely algorithmic, there is no additional
ICU data requirement (the current small-icu is just fine).
