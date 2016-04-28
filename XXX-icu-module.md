| Title  | ICU Module                  |
|--------|-----------------------------|
| Author | @jasnell                    |
| Status | DRAFT                       |
| Date   | 2016-04-20                  |

## Description

The ICU4C library that we use for internationalization contains a significant
array of additional functionality not currently exposed by the EcmaScript 402
standard. Some of this additional functionality would be useful to expose via
a new `'icu'` or (`'unicode'`) module.

## Interface

Initially, the `'icu'` module would provide methods for the following:

1. Character encoding detection. ICU includes code that is able to look at a
   stream of bytes and apply heuristics to detect the character encoding in
   use. This is not always an exact match but it does a reasonably good job.
   We can tune this detection to only look for the character encodings we
   support in Core (ascii, iso8859-1, utf8 and utf16-le). Two specific APIs
   would be exposed by the `'icu'` module for this capability:
   
```js
const icu = require('icu');

// Detect the encoding for a given buffer or string.
// Returns a string with the most likely match.
icu.detectEncoding(myBuffer);
icu.detectEncoding(myString);

// Detect the encoding for a given buffer or string.
// Returns an object whose keys are the detected
// encodings and whose values are a confidence value
// provided by ICU. The higher the confidence value,
// the better the match.
const encs = icu.detectEncodings(myBuffer);
console.log(encs);
 // Prints something like {'ascii': 90, 'utf8': 15}
```

This mechanism is useful when working with data that might be in multiple
character sets (such as filenames on Linux, or reading through multiple
files in a directory).

```
const data = getDataSomehow();
const buffer = Buffer.from(data, icu.detectEncoding(data));
```

2. One-Shot and Streaming Buffer re-encoding. ICU includes code for converting
   from one encoding to another. This is similar to what is provided by `iconv`
   but it is built in to ICU4C. The `'icu'` module would include converters for
   *only* the character encodings directly supported by core. Developers would
   continue to use `iconv` or `iconv-lite` for more exotic things.

```js
const icu = require('icu');

// One-shot conversion. Converts the entire Buffer in one go.
// Assumes that the Buffer is properly aligned on UFT-8 boundaries
const myBuffer = Buffer.from(getUtf8DataSomehow(), 'utf8');
const newBuffer = icu.reencode(myBuffer, 'utf8', 'ucs2');


// Streaming conversion
const convertStream = icu.createConverter('utf8', 'ucs2');
convertStream.on('data', (chunk) => {
  // chunk is a UTF-16 (ucs2) encoded buffer
});
// Writing UTF-8 data
convertStream.write(getUtf8DataSomehow());
```

Additional convenience methods would be attached to `Buffer.prototype`:

```
const myBuffer = Buffer.from(getUtf8DataShow(), 'uf8');
const newBuffer = myBuffer.reencode('utf8', 'ucs2');
```

Again, this would ONLY support the encodings for which we already have built-in
support in core (acsii, iso8859-1, utf8 and utf16). This does not expand the
encoding support in core so `iconv` and `iconv-lite` would still be necessary.

3. UTF-8 and UTF-16 aware `codePointAt()` and `charAt()` methods for `Buffer`.
   This one is pretty straightforward. They would return either the Unicode
   codepoint or the character at the given byte offset even if the byte offset
   is not on a UTF-8 or UTF-16 lead byte. These are intended to be symmetrical
   with `String.prototype.codePointAt()` and `String.prototype.charAt()`

```
const icu = require('icu');

const myBuffer = Buffer.from('a€bc', 'utf8');

console.log(icu.codePointAt(myBuffer, 1, 'utf8'));
// or
console.log(myBuffer.codePointAt(1, 'utf8'));

console.log(icu.charAt(myBuffer, 1, 'utf8'));
// or
console.log(myBuffer.charAt(1, 'utf8'));
```

4. UTF-16 and UTF-8 aware `slice()` for `Buffer`. This is similar to the
   existing `Buffer.prototype.slice()` except that, rather than byte offsets,
   the `start` and `end` are codepoint/character offsets This would make it
   symmetrical with `String.prototype.slice()` but for Buffers. The advantage
   is that this allows the Buffer to be sliced in a way that ensures proper
   alignment with UTF-8 or UTF-16 encodings.

```
const icu = require('icu');

const myBuffer = Buffer.from('a€bc', 'utf8');

icu.slice(myBuffer, 'utf8', 1, 2); // returns a Buffer with €
// or
Buffer.slice(1, 2, 'utf8'); // returns a Buffer with €
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
