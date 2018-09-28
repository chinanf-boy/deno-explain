## tool/run_rustc.py

启动rust编译器- `rustc`

### import

``` py
#!/usr/bin/env python
# 灵感源自
# https://fuchsia.googlesource.com/build/+/master/rust/build_rustc_target.py
# Copyright 2018 The Fuchsia Authors. All rights reserved.
# Use of this source code is governed by a BSD-style license that can be
# found in the LICENSE file.
import sys
import os
import argparse
import subprocess
import util
```

### fix_depfile

``` py
# 将depfile中主目标的路径更新为相对路径
# 根目录base_path, build_output_path
def fix_depfile(depfile_path, base_path, build_output_path):
    # 修复depfile与以前具有相同的mtime非常重要。
    # Ninja依靠它决定是否重建它所属的目标。
    stat = os.stat(depfile_path)
    with open(depfile_path, "r") as depfile:
        content = depfile.read()
    content_split = content.split(': ', 1)
    target_path = os.path.relpath(build_output_path, start=base_path)
    new_content = "%s: %s" % (target_path, content_split[1])
    with open(depfile_path, "w") as depfile:
        depfile.write(new_content)
    os.utime(depfile_path, (stat.st_atime, stat.st_mtime))

```

### main

命令行分析, 且启动`rustc`

``` py
def main():
    parser = argparse.ArgumentParser("编译一个 Rust crate")
    parser.add_argument(
        "--depfile",
        help="depfile应该存储的输出路径",
        required=False)
    parser.add_argument(
        "--output_file",
        help="输出文件应该存储的路径",
        required=False)
    args, rest = parser.parse_known_args()

    util.run(["rustc"] + rest, quiet=True)

    if args.depfile and args.output_file:
        fix_depfile(args.depfile, os.getcwd(), args.output_file)
```

- [x] [util.run](./util.md#run)

### __main__

命令行启用, 需要运行的函数

``` bash
if __name__ == '__main__':
    sys.exit(main())

```