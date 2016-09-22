| Title | Add types builtin module |
| ----- | ------------------------ |
| Author | @evanlucas |
| Status | DRAFT |
| Date | 2016-09-22 |

## Description

This proposal is to add a builtin module that provides a way to determine
whether the passed argument is of a particular type. Although this can be
done via a native addon [(v8is)](https://github.com/evanlucas/v8is), it
requires that a compiler be installed on the consumer's machine.

We can expose these methods using currently by using `v8::Value`.

The `util.is*` functions are currently deprecated in the documentation. This
proposal will add an upgrade path to those currently using those functions.

## Basic Examples

```js
const types = require('types');

console.log(types.isObject({}));     // true
console.log(types.isObject(null));   // false
console.log(types.isMap(new Map())); // true
console.log(types.isSet(new Set())); // true
console.log(types.isInt32(1));       // true
console.log(types.isUint32(-1));     // false
```

## Proof-of-concept

A proof-of-concept implementation can be found at:

  https://github.com/evanlucas/node/tree/types


## API

### `types.isArgumentsObject(arg)`

### `types.isArray(arg)`

### `types.isArrayBuffer(arg)`

### `types.isArrayBufferView(arg)`

### `types.isBoolean(arg)`

### `types.isBooleanObject(arg)`

### `types.isDataView(arg)`

### `types.isDate(arg)`

### `types.isFalse(arg)`

### `types.isFloat32Array(arg)`

### `types.isFloat64Array(arg)`

### `types.isFunction(arg)`

### `types.isGeneratorFunction(arg)`

### `types.isGeneratorObject(arg)`

### `types.isInt32(arg)`

### `types.isInt8Array(arg)`

### `types.isInt16Array(arg)`

### `types.isInt32Array(arg)`

### `types.isMap(arg)`

### `types.isMapIterator(arg)`

### `types.isNativeError(arg)`

### `types.isNull(arg)`

### `types.isNumber(arg)`

### `types.isNumberObject(arg)`

### `types.isObject(arg)`

### `types.isPromise(arg)`

### `types.isProxy(arg)`

### `types.isRegExp(arg)`

### `types.isSet(arg)`

### `types.isSetIterator(arg)`

### `types.isString(arg)`

### `types.isStringObject(arg)`

### `types.isSymbol(arg)`

### `types.isTrue(arg)`

### `types.isTypedArray(arg)`

### `types.isUint32(arg)`

### `types.isUint8Array(arg)`

### `types.isUint8ClampedArray(arg)`

### `types.isUint16Array(arg)`

### `types.isUint32Array(arg)`

### `types.isUndefined(arg)`

### `types.isWeakMap(arg)`

### `types.isWeakSet(arg)`
