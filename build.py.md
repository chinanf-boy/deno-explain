## build.py

运行`ninja`, 构建四个二进制文件`test.cc`-`test.rs`-`deno{''|'.exe'}`-`deno_ns{''|'.exe'}`

``` py
#!/usr/bin/env python
# Copyright 2018 the Deno authors. All rights reserved. MIT license.
from __future__ import print_function
import os
import sys
import third_party
from util import build_path, run

third_party.fix_symlinks()

ninja_args = sys.argv[1:] # ninja 构建时的参数
if not "-C" in ninja_args: # 获得GN 的生产 .ninja文件
    if not os.path.isdir(build_path()):
        print("Build directory '%s' does not exist." % build_path(),
              "Run tools/setup.py")
        sys.exit(1)
    ninja_args = ["-C", build_path()] + ninja_args

run([third_party.ninja_path] + ninja_args,
    env=third_party.google_env(),
    quiet=True)

```

> deno第三方包-V8-使用的构建系统是[Ninja](https://ninja-build.org/)，而 GN 是用来生成`.ninja`文件的工具。

- [x] [third_party > fix_symlinks](./tools/third_party.md#fix_symlinks)
- [x] [util > build_path](./tools/util.md#build_path)
- [x] [util > run](./tools/util.md#run)
- [x] [third_party > ninja_path](./tools/third_party.md#paths)
- [x] [third_party > google_env](./tools/third_party.md#google_env)

### 运行成功这一步之后

我们有了个, 测试版本的二进制文件deno

``` bash
# Run.
./out/debug/deno tests/002_hello.ts
```

但是, 在这一步, 此脚本仅仅是运行了一下`v8`的构建工具`ninja`, 就造出了融入多个语言的二进制文件

其中关于`ninja`和`gn`的使用配置大部分是需要去重新学习

### v8构建

- [ ] [gn 的使用]
- [ ] [ninja 的使用]