Virtual File System Requirements
================================

This document aims to list all the requirements of the Virtual File System.

# Supported

## Random access reads

The VFS must support random access reads just like any other real file system,
so that the read operations can be at least as fast as reading files from the
real file system.

## Symbolic links

This is critical for applications that want to use packages like [dugite][] that
attempt to download [Git executables][] that contain symlinks. Since
Electron's [ASAR][] does not support symlinks, including [dugite][] as a
dependency in an Electron app would expand every symlink into individual files,
thus significantly increase the package size which is not nice.

## Preserve the executable bit of the file permissions

It is important to preserve the executable bit of the file permissions, so that
it is possible for the single-executable to be able to execute only executable
files. Other than that, all the bundled files would be readable and none will be
writable.

## Optional data compression

As an application grows, bundling all the source code, dependencies and static
assets into a single file without compression would quickly reach the maximum
segment / file (depending on the embedding approach) size limit imposed by the
single executable file format / OS. A solution to this problem would be to
minify the JS source files but that might not be enough for other kinds of
files, so supporting data compression seems to be a better solution.

## Preserve file-hierarchy information

A filesystem is incomplete without this because there's no way for the
single-executable to be able to access nested file paths.

## No interference with valid paths in the file system

If the bundled files in the VFS correspond to certain paths that already exist
in the real file system, that will be very confusing, so it should use such
paths that cannot be used by existing files.

Pkg uses [`/snapshot`](https://github.com/vercel/pkg#snapshot-filesystem) as the
prefix for all the embedded files. This is confusing if `/snapshot` is an
existing directory on the file system. Docker workflows routinely copy files to,
and run things at, the root of the filesystem, so following that approach too
would run into the same problem.

Boxednode allows users to enter a [namespace](https://github.com/mongodb-js/boxednode/blob/6326e3277469e8cfe593616a0ed152600a5f9045/README.md?plain=1#L69-L72)
and uses it like so:
```js
  // Specify the entrypoint target name. If this is 'foo', then the resulting
  // binary will be able to load the source file as 'require("foo/foo")'.
  // This defaults to the basename of sourceFile, e.g. 'bar' for '/path/bar.js'.
  namespace?: string;
```

It might be better to use the single executable path as the base path for the
files in the VFS, i.e., if the executable has `/a/b/sea` as the path and the VFS
contains a file named `file.txt`, it would be accessible by the application
using `/a/b/sea/file.txt`. This approach is similar to how Electron's [ASAR][]
works, i.e., if the application asar is placed in `/a/b/app.asar`, the
embedded `file.txt` file would use `/a/b/app.asar/file.txt` as the path.

## Globbing

`fs.statSync(process.execPath).isDirectory()` will return `false` and
`fs.statSync(process.execPath).isFile()` will return `true`. That way, if code
within the single-executable does naive globbing using an off-the-shelf glob
library, VFS file paths, which are considered internal to the single-executable,
would not interfere with the actual file paths.

## Cross-platform tooling

The tooling required for archiving / extracting files into / from the VFS must
be available on all the [platforms supported by Node.js][].

## File path contents

Should not limit the size or the character contents of the file paths to stay as
close as possible to what a real file system provides.

## Case Sensitive

From Yarn's experience with zip, forcing case sensitivity within the archives
didn't break anything, improved consistency. By contrast, making the code case
insensitive would have increased the complexity, worsened the runtime
performance, increased the attack surface, for a use case that virtually no-one
cares about. Hence, the paths in the VFS will be case sensitive.

## Dynamic imports and requires

`require(require.resolve('./file.js'))` should work for files that are on the
real file system and the VFS.

## VFS path manipulation as strings and URL objects

If someone proposes that the VFS exist at a `vfs-file://` prefix, then this
might become an issue. `fs` APIs accept `URL` objects, but this means code in
(transitive) dependencies which assumes all native paths are strings may fail
when passed `URL` objects. Perhaps a (transitive) dependency uses
`require.resolve()`.

Using something like `vfs-file://` might be a potential solution for placing the
VFS contents somewhere that has no interference with valid paths in the file
system.

## Interaction with Native Addons

TODO: Still under discussion in https://github.com/nodejs/single-executable/discussions/29.

# Not supported

## No need for supporting write operations

Since the VFS is going to be embedded into the single-executable and also
protected by codesigning, making changes to the contents of the VFS should
invalidate the signature and crash the application if run again. Hence, no write
operation needs to be supported.

# Optionally support

## Increase locality of related files

For performance reasons.

## Format implementation in multiple languages

We want this format to already have implementation in *multiple* languages (not
just JS, since not all tools used in the JS ecosystem are written in JS), all
ideally production-grade and well-maintained.

## Consensus with third-party tools on building native integrations

We want this format to be consensual enough that third-party tools (VSCode,
emacs, ...) won't object to build native integrations with it (for instance,
Esbuild recently added zip support to integrate w/ Yarn's zip installs; it would
have been a much harder sell if Yarn had used a custom-made format).

[ASAR]: https://github.com/electron/asar
[Git executables]: https://github.com/desktop/dugite-native/releases/
[dugite]: https://www.npmjs.com/package/dugite
[platforms supported by Node.js]: https://github.com/nodejs/node/blob/main/BUILDING.md#supported-platforms
