## format.py

### import 
``` py
#!/usr/bin/env python
from glob import glob
import os
from third_party import third_party_path, fix_symlinks, google_env, clang_format_path
from util import root_path, run, find_exts

```

### format

格式化代码, 有五种那么多

0. clang_format - `.cc`
1. gn format - `.gn`
2. yapf - `python`
3. prettier - `js,ts`
4. rustfmt - `rust`

- [x] [third_party > ix_symlinks](./tools/third_party.md#fix_symlinks)

``` py
fix_symlinks()

prettier = os.path.join(third_party_path, "node_modules", "prettier",
                        "bin-prettier.js")
tools_path = os.path.join(root_path, "tools")
rustfmt_config = os.path.join(tools_path, "rustfmt.toml")

os.chdir(root_path)

run([clang_format_path, "-i", "-style", "Google"] +
    find_exts("libdeno", ".cc", ".h")) # 0

for fn in ["BUILD.gn", ".gn"] + find_exts("build_extra", ".gn", ".gni"):
    run(["third_party/depot_tools/gn", "format", fn], env=google_env()) # 1

# 在 tools 目录中, 我们使用 `glob()` 替代使用 `find_exts()`, 因为:
#   * 在 Windows, `os.walk()` (命名为 `find_exts()`) 跟随快捷连接(方式).
#   * 这个 tools 目录 包含 快捷连接 'clang', 指向
#     'third_party/v8/tools/clang'目录, 这个目录包含许多 .py 文件.
#   * 这些 第三方的 python文件 不应该被 格式代码.
#   * 这个 tools文件夹并没有子目录, 所以 `glob()`是有效的.
# TODO(ry) Install yapf in third_party.
run(["yapf", "-i"] + glob("tools/*.py") + find_exts("build_extra", ".py")) # 2

run(["node", prettier, "--write"] + find_exts("js/", ".js", ".ts") + # 3
    find_exts("tests/", ".js", ".ts") +
    ["rollup.config.js", "tsconfig.json", "tslint.json"])

# 要求 rustfmt 0.8.2 (命令行参数 与 上个版本不同)
run(["rustfmt", "--config-path", rustfmt_config] + find_exts("src/", ".rs")) # 4

```

- third_party
    - [third_party_path](./tools/third_party.md#paths)
    - [google_env](./tools/third_party.md#google_env)
    - [clang_format_path](./tools/third_party.md#paths)
- util
    - [root_path](./tools/util.md#import)
    - [run](./tools/util.md#run)
    - [find_exts](./tools/util.md#find_exts)