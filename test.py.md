## test.py

### import

``` py
#!/usr/bin/env python
# 运行 完整 测试 用例.
# 使用: ./tools/test.py out/Debug
import os
import sys
from check_output_test import check_output_test
from setup_test import setup_test
from util import executable_suffix, run, build_path
from unit_tests import unit_tests
from util_test import util_test
import subprocess
import http_server
```

### check_exists

文件是否存在

```py
def check_exists(filename):
    if not os.path.exists(filename):
        print "Required target doesn't exist:", filename
        print "Run ./tools/build.py"
        sys.exit(1)

```

### main

#### 1. web 测试

```py
def main(argv):
    if len(argv) == 2:
        build_dir = sys.argv[1]
    elif len(argv) == 1:
        build_dir = build_path()
    else:
        print "Usage: tools/test.py [build_dir]"
        sys.exit(1)

    http_server.spawn()

    # 网络工具 测试...
    setup_test()
    util_test()

```

- [x] [http_server](./tools/http_server.md)
- [x] [setup_test](./tools/setup_test.md)
- [x] [util_test](./tools/util_test.md)

#### 2. 构建的产物测试

``` py
    # executable_suffix 若是 windows 就为`.exe`后缀
    test_cc = os.path.join(build_dir, "test_cc" + executable_suffix)
    check_exists(test_cc) # 测试 c 文件 TODO 不太清楚测试内容
    run([test_cc])

    test_rs = os.path.join(build_dir, "test_rs" + executable_suffix)
    check_exists(test_rs) # 测试 rust 文件 TODO 不太清楚测试内容
    run([test_rs])

    deno_exe = os.path.join(build_dir, "deno" + executable_suffix)
    check_exists(deno_exe)
    unit_tests(deno_exe)

    check_exists(deno_exe)
    check_output_test(deno_exe)

    deno_ns_exe = os.path.join(build_dir, "deno_ns" + executable_suffix)
    check_exists(deno_ns_exe)
    check_output_test(deno_ns_exe)

```

- [x] [executable_suffix](./util.md#import)
- [x] [check_exists](#check_exists) 存在文件吗
- [x] [util_tests](./tools/util_tests.md)
- [x] [check_output_test](./tools/check_output_test.md)

### __main__

``` py
if __name__ == '__main__':
    sys.exit(main(sys.argv))
```
