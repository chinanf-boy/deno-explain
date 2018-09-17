## check_output_test

测试整个`根/test`目录下文件

### import

``` py
#!/usr/bin/env python
# 给定一个deno可执行文件，该脚本为执行文件运行多个集成测试
# 测试存储在 //tests/ 中，每个脚本都有相应的
# .out文件，指定stdout应该是什么。
#
# Usage: check_output_test.py [path to deno executable]
import os
import sys
import subprocess
from util import pattern_match, parse_exit_code

root_path = os.path.dirname(os.path.dirname(os.path.realpath(__file__)))
tests_path = os.path.join(root_path, "tests")
```

### check_output_test

从`根/tests`中, 用deno文件执行其中的`名称.js/ts`文件, 得到的结果与`同名.out`的内容相当

0. 循环测试文件
1. 整理 测试
2. 子进程运行测试
3. 若以下情况, 退出测试
    - 3.1 运行中, 预期成功, 但运行错误
    - 3.2 预期退出代码不一致
    - 3.3 输出没有匹配的模式

```py
def check_output_test(deno_exe_filename):
    assert os.path.isfile(deno_exe_filename)
    outs = sorted([
        filename for filename in os.listdir(tests_path)
        if filename.endswith(".out")
    ])
    assert len(outs) > 1
    tests = [(os.path.splitext(filename)[0], filename) for filename in outs]
    for (script, out_filename) in tests: # 0
        script_abs = os.path.join(tests_path, script)
        out_abs = os.path.join(tests_path, out_filename)
        with open(out_abs, 'r') as f:
            expected_out = f.read()
        cmd = [deno_exe_filename, script_abs, "--reload"]
        expected_code = parse_exit_code(script)
        print " ".join(cmd) # 1.
        actual_code = 0
        try:
            actual_out = subprocess.check_output(cmd, universal_newlines=True) # 2.
        except subprocess.CalledProcessError as e:
            actual_code = e.returncode
            actual_out = e.output
            if expected_code == 0:
                print "Expected success but got error. Output:"  # 3.1
                print actual_out
                sys.exit(1)

        if expected_code != actual_code: # 3.2 预期码不一致
            print "Expected exit code %d but got %d" % (expected_code,
                                                        actual_code)
            print "Output:"
            print actual_out
            sys.exit(1)

        if pattern_match(expected_out, actual_out) != True: # 3.3 找不到相同模式
            print "Expected output does not match actual."
            print "Expected: " + expected_out
            print "Actual:   " + actual_out
            sys.exit(1)

```

- [x] [parse_exit_code](./util.md#parse_exit_code)
- [x] [pattern_match](./util.md#pattern_match)

### main

```py
def main(argv):
    check_output_test(argv[1])

```

### __main__

```py
if __name__ == '__main__':
    sys.exit(main(sys.argv))
```