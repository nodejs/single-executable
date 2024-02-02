Existing SEA Solutions
======================

This document aims to exhaustively list and categorize every known solution for
packaging Node.js applications as standalone executables.

| Name       | Company      | Repository                              | Point of Contact |
|------------|--------------|-----------------------------------------|------------------|
| pkg        | Vercel       | https://github.com/vercel/pkg (archived)<br/>https://github.com/yao-pkg/pkg | @jesec<br>@robertsLando         |
| boxednode  | MongoDB      | https://github.com/mongodb-js/boxednode | @addaleax        |
| nexe       | -            | https://github.com/nexe/nexe            | N/A              |
| node-sea   | -            | https://github.com/arcanis/node-sea     | @arcanis         |
| encloseJs | - | https://github.com/igorklopov/enclose (archived) | @igorklopov |

Virtual File System Implementations
-----------------------------------

| Name  | Project  | Reference                                                                                          | Point of Contact |
|-------|----------|----------------------------------------------------------------------------------------------------|------------------|
| ASAR  | Electron | <ul><li>[ASAR format]</li><li>[monkey patching of `fs` in Electron to read from an ASAR]</li></ul> | [`@zcbenz`]      |
| ZipFS | Yarn     | [ZipFS]                                                                                            | @arcanis         |

[ASAR format]: https://github.com/electron/asar
[`@zcbenz`]: https://github.com/zcbenz
[monkey patching of `fs` in Electron to read from an ASAR]: https://github.com/electron/electron/blob/06a00b74e817a61f20e2734d50d8eb7bc9b099f6/lib/asar/fs-wrapper.ts
[ZipFS]: https://github.com/yarnpkg/berry/blob/master/packages/yarnpkg-libzip/sources/ZipFS.ts

Miscellaneous Related Tooling
-----------------------------

| Name           | Notes                                      |  
|----------------|--------------------------------------------|
| [`qjsc`][qjsc] | Compiles Javascript sources to executables with no external dependency. |

[qjsc]: https://bellard.org/quickjs/quickjs.html#qjsc-compiler

Did we miss any or got any details wrong? [Help us out by sending a
PR!](https://github.com/nodejs/single-executable/edit/main/docs/existing-solutions.md)
