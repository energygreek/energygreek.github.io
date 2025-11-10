---
comments: true
date: 2025-02-28
tags: [clang, dev-tool]
---

# ways of generating compile command json file
clangd is programming language server of clang, requires a compile_commands.json file located in source code directory for the best. I recommend intercept-build for generating the file, because it is included in clang-tools package when compiling via make and makefile.

* intercept-build make -B
* bear make -B
* compiledb make -B
* meson, cmake
* [codechecker](https://github.com/Ericsson/codechecker/#Install-guide)

or use bash script
```sh
make --always-make --dry-run \
 | grep -wE 'gcc|g\+\+' \
 | grep -w '\-c' \
 | jq -nR '[inputs|{directory:".", command:., file: match(" [^ ]+$").string[1:]}]' \
 > compile_commands.json
```


## reference
https://releases.llvm.org/11.0.1/tools/clang/docs/JSONCompilationDatabase.html
https://gist.github.com/gtors/effe8eef7dbe7052b22a009f3c7fc434
https://stackoverflow.com/questions/21134120/how-to-turn-makefile-into-json-compilation-database