## util_tests

测试 二进制文件`deno{''|'.exe'}`

### import
``` py
#!/usr/bin/env python
from util import run
import sys

```

- [x] [run](./util.md#run)

### util_tests

用构建出来的二进制文件,测试ts文件, 拿到结果字符进行对比

```py
# 我们想要在deno中测试许多具有不同权限设置的行为操作.
# 这些测试可以指定他们期望的权限，
# 它将一个特殊的字符串如“permW1N0”附加到测试名称的末尾。
# 这里我们运行几个具有不同权限的deno副本，
# 通过特殊字符串进行过滤测试。 permW0N0表示允许写入但不允许网络。
# 有关更多详细信息，请参阅 js/test_util.ts。
def unit_tests(deno_exe):
    run([deno_exe, "--reload", "js/unit_tests.ts", "permW0N0E0"])
    run([
        deno_exe, "--reload", "js/unit_tests.ts", "permW1N0E0", "--allow-write"
    ])
    run([
        deno_exe, "--reload", "js/unit_tests.ts", "permW0N1E0", "--allow-net"
    ])
    run([
        deno_exe, "--reload", "js/unit_tests.ts", "permW0N0E1", "--allow-env"
    ])
    run([
        deno_exe,
        "--reload",
        "js/unit_tests.ts",
        "permW1N1E1",
        "--allow-write",
        "--allow-net",
        "--allow-env",
    ])

```

- [ ] [js/unit_tests.ts](./js/unit_tests.md)

### __main__

```py
if __name__ == '__main__':
    if len(sys.argv) < 2:
        print "Usage ./tools/unit_tests.py out/debug/deno"
        sys.exit(1)
    unit_tests(sys.argv[1])
```