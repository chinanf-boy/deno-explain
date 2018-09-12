## util

### import

``` py
# 版权2018年 Deno 作者所有. MIT许可证.
import os
import re
import shutil
import stat
import sys
import subprocess

executable_suffix = ".exe" if os.name == "nt" else ""
root_path = os.path.dirname(os.path.dirname(os.path.realpath(__file__)))

```

### make_env

返回新的`env`副本,并被`merge_env[key]`-相同key赋值

```py
def make_env(merge_env={}, env=None):
    if env is None:
        env = os.environ
    env = env.copy()
    for key in merge_env.keys():
        env[key] = merge_env[key]
    return env

```

### run

相当于在`shell`中输入, 一串代码, 运行

```py
def run(args, quiet=False, cwd=None, env=None, merge_env={}):
    args[0] = os.path.normpath(args[0])
    if not quiet:
        print " ".join(args)
    env = make_env(env=env, merge_env=merge_env)
    shell = os.name == "nt"  # 通过shell运行以使.bat/.cmd文件正常工作。
    rc = subprocess.call(args, cwd=cwd, env=env, shell=shell)
    if rc != 0:
        sys.exit(rc)

```

### run_output

```py
def run_output(args, quiet=False, cwd=None, env=None, merge_env={}):
    args[0] = os.path.normpath(args[0])
    if not quiet:
        print " ".join(args)
    env = make_env(env=env, merge_env=merge_env)
    shell = os.name == "nt"  # 通过shell运行以使.bat / .cmd文件正常工作。
    return subprocess.check_output(args, cwd=cwd, env=env, shell=shell)

```

### remove_and_symlink

```py
def remove_and_symlink(target, name, target_is_dir=False):
    try:
        # 在Windows上，只能使用 rmdir() 
        # 删除 空目录
        if os.name == "nt" and os.path.isdir(name):
            os.rmdir(name) 
        else:
            os.unlink(name) # 删除文件, 若name是目录,出错
    except: # 其他系统, 目录不做事
        pass

    symlink(target, name, target_is_dir)
```

- [x] [symlink](#symlink)

### symlink

建立, 软连接 - (快捷方式)

1. `windows` 的 快捷链接文件创建的 各种测与验
2. 其他系统

> 不想看

```py
def symlink(target, name, target_is_dir=False):
    
    if os.name == "nt": # windows
        from ctypes import WinDLL, WinError, GetLastError
        from ctypes.wintypes import BOOLEAN, DWORD, LPCWSTR

        kernel32 = WinDLL('kernel32', use_last_error=False)
        CreateSymbolicLinkW = kernel32.CreateSymbolicLinkW
        CreateSymbolicLinkW.restype = BOOLEAN
        CreateSymbolicLinkW.argtypes = (LPCWSTR, LPCWSTR, DWORD)

        # 文件类型符号链接只能使用反斜杠作为分隔符。
        target = os.path.normpath(target)

        # 如果符号链接指向目录，则需要具有相应的符号
        # 标志设置，否则将创建链接但它将无法正常工作。
        if target_is_dir:
            type_flag = 0x01  # SYMBOLIC_LINK_FLAG_DIRECTORY
        else:
            type_flag = 0

        # 在Windows 10之前，创建符号链接需要管理员权限。
        # 从Win 10开始，有一个标志允许任何人创建它们。
        # 最初，尝试使用此标志。
        unpriv_flag = 0x02  # SYMBOLIC_LINK_FLAG_ALLOW_UNPRIVILEGED_CREATE
        r = CreateSymbolicLinkW(name, target, type_flag | unpriv_flag)

        # 如果ERROR_INVALID_PARAMETER失败，请在没有的情况下再试一次
        # 'allow unprivileged create'标志。
        if not r and GetLastError() == 87:  # ERROR_INVALID_PARAMETER
            r = CreateSymbolicLinkW(name, target, type_flag)

        # 即使在第二次尝试后仍然失败。
        if not r:
            raise WinError()
    else:
        os.symlink(target, name)
```

### touch

```py
def touch(fname):
    if os.path.exists(fname):
        os.utime(fname, None)
    else:
        open(fname, 'a').close()

```

### find_exts

```py
# 递归搜索某些扩展名的文件。
# （python 2.7中不存在递归glob。）
def find_exts(directory, *extensions):
    matches = []
    for root, dirnames, filenames in os.walk(directory):
        for filename in filenames:
            for ext in extensions:
                if filename.endswith(ext):
                    matches.append(os.path.join(root, filename))
                    break
    return matches

```

###  rmtree

```py
# Python相当于`rm -rf`。
def rmtree(directory):
    # 在Windows上，shutil.rmtree() 不会删除具有只读位的文件。
    # Git创建了一些文件。 'onerror'回调处理那些。
    def rm_readonly(func, path, _):
        os.chmod(path, stat.S_IWRITE)
        func(path)

    shutil.rmtree(directory, onerror=rm_readonly)

```

### build_mode

```py
def build_mode(default="debug"):
    if "DENO_BUILD_MODE" in os.environ:
        return os.environ["DENO_BUILD_MODE"]
    else:
        return default


# 例如。 “出/调试”
def build_path():
    if "DENO_BUILD_PATH" in os.environ:
        return os.environ["DENO_BUILD_PATH"]
    else:
        return os.path.join(root_path, "out", build_mode())

```

### pattern_match

```py
# 如果预期与实际输出匹配，则返回True，允许变化
# 从实际预期的通配符（例如匹配/.*/）
def pattern_match(pattern, string, wildcard="[WILDCARD]"):
    if len(pattern) == 0:
        return string == 0
    if pattern == wildcard:
        return True

    parts = str.split(pattern, wildcard)

    if len(parts) == 1:
        return pattern == string

    if string.startswith(parts[0]):
        string = string[len(parts[0]):]
    else:
        return False

    for i in range(1, len(parts)):
        if i == (len(parts) - 1):
            if parts[i] == "" or parts[i] == "\n":
                return True
        found = string.find(parts[i])
        if found < 0:
            return False
        string = string[(found + len(parts[i])):]

    return len(string) == 0

```

### parse_exit_code

```py
def parse_exit_code(s):
    codes = [int(d or 1) for d in re.findall(r'error(\d*)', s)]
    if len(codes) > 1:
        assert False, "doesn't support multiple error codes."
    elif len(codes) == 1:
        return codes[0]
    else:
        return 0

```