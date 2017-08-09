| Title  | Implement --extension flag  |
|--------|-----------------------------|
| Author | @bmeck                      |
| Status | DRAFT                       |
| Date   | 2017-08-09T10:00:00-05:00   |

## Description

The Node CLI currently accepts input in both filename (`node app.js`) and opaque text stream form (`node -e '123'` and `node <app.js`). This behavior always results in running the stream as a CommonJS goal. With the upcoming usage of ESM and WASM there will be cases where developers wish to pipe other forms of text to the CLI.

This proposal is to implement an `--entry-extension` CLI flag for the cases which may want to use a stream that does not have a file extension.

## Example

```sh
node --entry-extension=js <app.js
node --entry-extension=js -e $(cat <app.js)
node --entry-extension=wasm <app.wasm
node --entry-extension=wasm -e $(cat <app.wasm)
node --entry-extension=mjs <app.mjs
node --entry-extension=mjs -e $(cat <app.mjs)
```

## Alternate Possibilities

Some alternate possibilities exist that might be relevant. `--entry-url` for example could be used to provide an `import.meta.url` properly while also providing the pathname that contains an extension. However, since URLs are not mandated to have file extensions this might be for nothing. Applications can also access `process.cwd()` to recreate similar data to `import.meta.url`.

## APIs

No new runtime APIs are to be added.

## Future considerations

Some upcoming file formats are also proposed such as [WebPackage](https://github.com/WICG/webpackage) that would also benefit from this.
