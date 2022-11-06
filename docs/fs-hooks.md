# FS hooks

This provides a way for userland code to overload the read-only operations on a virtual file system.
The internals of the `fs` API will be changed to make calls to these hooks.

## `load(path, type)`

* `path`: [`<string>`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#String_type) The requested file path.
* `type`: `<type.FileType | type.DirectoryType | type.NotFound | undefined>` The possible file types. If `undefined`, retry with the original `fs` API.
* Returns:
  * `contents`: [`<Buffer>`](https://nodejs.org/api/buffer.html#class-buffer) The contents of the file path.
  * `type` : `<type.File | type.Directory | type.NotFound | undefined>` The type of the file. If `undefined`, retry with the original `fs` API.
  * `mode`: [`<integer>`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Number_type) The file mode bit mask.

### Example usage

<details>
  
<summary>Exposing VFS code to JS via <code>process._singleExecutableApplicationCode</code></summary>

The embedded VFS can be loaded and exposed to JS through `process._singleExecutableApplicationCode` in
https://github.com/nodejs/node/blob/962b9abbe3126202d92d0f8a03adad9f51836dbb/src/node.cc#L320.

```cc
// Refs: https://github.com/postmanlabs/postject/blob/a9d8f3de44129d105208c9ad550c549e61ff82be/postject-api.h
#include <postject-api.h>

...

  size_t single_executable_application_size = 0;
  const char* single_executable_application_code =
      static_cast<const char*>(postject_find_resource(
          "NODE_JS_CODE", &single_executable_application_size, nullptr));
  if (single_executable_application_code != nullptr) {
    Isolate* isolate = env->isolate();
    Local<v8::Context> context = env->context();
    Local<Value> buffer =
        Buffer::New(
            isolate,
            const_cast<char*>(single_executable_application_code),
            single_executable_application_size,
            [](char* data, void* hint) {},
            nullptr)
            .ToLocalChecked();
    READONLY_PROPERTY(
        env->process_object(), "_singleExecutableApplicationCode", buffer);
    return StartExecution(env, "internal/main/single_executable_application");
  }

...
```
</details>


```mjs  
// asar-loader.mjs

// See https://github.com/electron/node-chromium-pickle-js/blob/238613902005ebac2481f295db59e980f8a236ac/lib/pickle.js
import Pickle from './pickle.mjs';

const headerSizeLength = 8;

function readHeader (content) {
  const sizeBuf = content.subarray(0, headerSizeLength)
  const sizePickle = new Pickle(sizeBuf)
  const size = sizePickle.createIterator().readUInt32()

  const headerBuf = content.subarray(headerSizeLength, headerSizeLength + size)
  const headerPickle = new Pickle(headerBuf)
  const header = headerPickle.createIterator().readString()

  return { headerString: header, header: JSON.parse(header), headerSize: size }
}

const asarBuffer = process._singleExecutableApplicationCode;
const asar = readHeader(asarBuffer);

const prefix = process.execPath;

export function load(path, { FileType, DirectoryType, NotFound }) {
  if (!path.startsWith(prefix)) {
    // File not present inside the VFS; retry operation with original fs.
    return;
  }
  
  const request = path.slice(prefix.length + 1).split('/'));
  
  let obj = asar.header;
  for (let i = 0; i < request.length; ++i) {
    obj = obj.files[request[i]];
    if (!obj) {
      // File not found.
      return { type: NotFound };
    }
  }

  if (obj.files && (!obj.offset || !obj.size)) {
    // Directory found.
    return {
      type: DirectoryType,
      contents: obj.files,
      mode: 0o755 // ASAR doesn't support file modes yet, so this returns a default value.
    };
  }

  const { offset, size } = obj;
  const pre = headerSizeLength + asar.headerSize;
  const start = pre + Number(offset);
  const end = start + Number(size);
  const contents = asarBuffer.subarray(start, end);

  return {
    type: FileType,
    contents: asarBuffer.subarray(start, end),
    mode: 0o644 // ASAR doesn't support file modes yet, so this returns a default value.
  };
}
```

The hook can be used by passing the file path through the `--experimental-fs-hook` CLI flag.

```sh
node --experimental-fs-hook ./asar-loader.mjs
```
