## third_party.py

此脚本包含辅助函数以使用[third_party]()子repo. 

### import

```py
#!/usr/bin/env python
# 此脚本包含辅助函数以使用third_party子参数。

import os
import sys
from os import path
from util import find_exts, make_env, remove_and_symlink, rmtree, root_path, run
```
 
- [ ] [find_exts ](./util.md#find_exts)
- [ ] [remove_and_symlink ](./util.md#remove_and_symlink)
- [ ] [rmtree ](./util.md#rmtree)
- [ ] [root_path ](./util.md#root_path)
- [x] [run](./util.md#run)

### root

``` py
# Helper函数，返回repo根目录下的子路径的完整路径。
def root(*subpath_parts):
    return path.normpath(path.join(root_path, *subpath_parts))

```

### tp

``` py
# Helper函数，返回 third_party 中 文件/目录 的完整路径。
def tp(*subpath_parts):
    return root("third_party", *subpath_parts)
```

### paths

``` py
third_party_path = tp() # 第三方包目录
depot_tools_path = tp("depot_tools")  # 三方包/depot_tools
rust_crates_path = tp("rust_crates") # 三方包/rust_crates
gn_path = tp(depot_tools_path, "gn") # 三方包/depot_tools/gn
clang_format_path = tp(depot_tools_path, "clang-format") # 三方包/depot_tools/clang-format
ninja_path = tp(depot_tools_path, "ninja") # 三方包/depot_tools/ninja
```

### google_env

创建或修改环境,以使其与符合各种谷歌工具（gn，gclient等）的期望。

``` py
def google_env(env=None, merge_env={}, depot_tools_path=depot_tools_path):
    env = make_env(env=env, merge_env=merge_env)
    # 在Python之前，Depot_tools要位于PATH中。
    path_prefix = depot_tools_path + os.path.pathsep
    if not env['PATH'].startswith(path_prefix):
        env['PATH'] = path_prefix + env['PATH']

    if os.name == "nt":  # 仅限 Windows的环境调整。
        # 我们没有使用Google的内部基础架构。
        if os.name == "nt" and not "DEPOT_TOOLS_WIN_TOOLCHAIN" in env:
            env["DEPOT_TOOLS_WIN_TOOLCHAIN"] = "0"

        # 这'setup_toolchain.py'脚本可以很好地找到 Windows
        # SDK。不幸的是，如果设置了以下任何环境变量
        # （正如vcvarsall.bat通常会这样），`setup_toolchain`也会吸收它们，
        # 将多个相同的 -imsvc<path> 项添加到 CFLAGS。
        # 这个小变化对编译器输出没有影响，但它
        # 能使 Nanji重建一切，并导致 sccache缓存不起作用。
        # TODO (piscisaureus):解决这个问题。
        env["INCLUDE"] = ""
        env["LIB"] = ""
        env["LIBPATH"] = ""

    return env

```

- [x] [make_env ](./util.md#make_env) 

> 返回新的`env`副本,并被`merge_env[key]`-相同key赋值

### fix_symlinks

制造各个快捷连接

``` py
def fix_symlinks():
    # 确保third_party目录存在。
    try:
        os.makedirs(third_party_path) # 创建 third_party_path 目录
    except:
        pass

    # 将符号链接设置为生成在根目录中的Yarn元数据。
    remove_and_symlink("../package.json", tp("package.json"))

    # TODO（ry）是否可以删除这些符号链接 ？
    remove_and_symlink("v8/third_party/googletest", tp("googletest"), True)
    remove_and_symlink("v8/third_party/jinja2", tp("jinja2"), True)
    remove_and_symlink("v8/third_party/llvm-build", tp("llvm-build"), True)
    remove_and_symlink("v8/third_party/markupsafe", tp("markupsafe"), True)

    # 在Windows上，如果符号链接的目标是在不同的git项目，git不会创建正确类型的符号链接
    # 。在这里，我们修复了存在的符号链接
    # 软连接在根目录中，而他们的目标是在 third_party 项目中.
    remove_and_symlink("third_party/node_modules", root("node_modules"), True)
    remove_and_symlink("third_party/v8/build", root("build"), True)
    remove_and_symlink("third_party/v8/buildtools", root("buildtools"), True)
    remove_and_symlink("third_party/v8/build_overrides",
                       root("build_overrides"), True)
    remove_and_symlink("third_party/v8/testing", root("testing"), True)
    remove_and_symlink("../third_party/v8/tools/clang", root("tools/clang"),
                       True)

```

- [x] [util > remove_and_symlink](./util.md#remove_and_symlink) 

> 如果可能,删除第二个路径参数的文件/目录, 然后建起第一个路径到第二路径的快捷链接(软连接)

## 运行各自的下载工具

### run_yarn

运行Yarn以安装JavaScript依赖项。

``` py
def run_yarn():
    run(["yarn", "install"], cwd=third_party_path)
```

### run_cargo

运行Cargo以安装Rust依赖项。

```py
def run_cargo():
    # 删除货物索引锁定文件;看来货物本身并没有这样做。
    # 如果锁定文件最终出现在git仓库中，那么它将使每个人的货物挂起
    # 尝试运行sync_third_party的其他人。
    def delete_lockfile():
        lockfiles = find_exts(
            path.join(rust_crates_path, "registry/index"), '.cargo-index-lock')
        for lockfile in lockfiles:
            os.remove(lockfile)

    # 删除索引锁定文件，以防有人意外签入。
    delete_lockfile()

    run(["cargo", "fetch", "--manifest-path=" + root("Cargo.toml")],
        cwd=third_party_path,
        merge_env={'CARGO_HOME': rust_crates_path})

    # 再次删除锁定文件，使其不会在git仓库中结束。
    delete_lockfile()


```

### run_gclient_sync

运行`gclient`以安装其他依赖项。

```py
def run_gclient_sync():
    # Depot_tools通常会尝试自我更新，因为这会失败
    # 它没有从它自己的git存储库中检出;然后gclient会尝试
    # 解决问题而不是成功，我们最终将陷入巨大的混乱。
    # 要解决此问题，我们将`depot_tools`目录重命名为
    # 首先是`{root_path} / depot_tools_temp`，我们设置DEPOT_TOOLS_UPDATE = 0 in
    # 环境因此depot_tools不会尝试自我更新。
    # 由于depot_tools列在.gclient_entries中，gclient将安装一个
    # `third_party / depot_tools`中的新副本。
    # 如果一切顺利，我们之后会删除depot_tools_temp目录。
    depot_tools_temp_path = root("depot_tools_temp")

    # 将depot_tools重命名为depot_tools_temp。
    try:
        os.rename(depot_tools_path, depot_tools_temp_path)
    except:
        # 如果重命名失败，并且depot_tools_temp目录已存在，
        # 假设它仍然存在，因为先前的run_gclient_sync（）调用
        # 在我们有机会移除临时目录之前，中途失败了。
        # 我们将使用已存在的临时目录中的任何内容。
        # 如果没有，用户可以通过手动删除临时目录来恢复。
        if path.isdir(depot_tools_temp_path):
            pass
        else:
            raise

    args = [
        "gclient", "sync", "--reset", "--shallow", "--no-history", "--nohooks"
    ]
    envs = {
        'DEPOT_TOOLS_UPDATE': "0",
        'GCLIENT_FILE': root("gclient_config.py")
    }
    env = google_env(depot_tools_path=depot_tools_temp_path, merge_env=envs)
    run(args, cwd=third_party_path, env=env)

    # 删除depot_tools_temp目录，但不能在验证之前删除
    # gclient确实安装了一个新的副本。
    # 还要检查`{depot_tools_temp_path} / gclient.py`是否存在，所以输入错字
    # 这个剧本不会意外地吹灭某人的家庭目录。
    if (path.isdir(path.join(depot_tools_path, ".git"))
            and path.isfile(path.join(depot_tools_path, "gclient.py"))
            and path.isfile(path.join(depot_tools_temp_path, "gclient.py"))):
        rmtree(depot_tools_temp_path)

```

## 从谷歌存储中下载文件

### download_from_google_storage

从Google存储中下载给定的项目。

``` py
def download_from_google_storage(item, bucket):
    if sys.platform == 'win32':
        sha1_file = "v8/buildtools/win/%s.exe.sha1" % item
    elif sys.platform == 'darwin':
        sha1_file = "v8/buildtools/mac/%s.sha1" % item
    elif sys.platform.startswith('linux'):
        sha1_file = "v8/buildtools/linux64/%s.sha1" % item

    run([
        "python",
        tp('depot_tools/download_from_google_storage.py'),
        '--platform=' + sys.platform,
        '--no_auth',
        '--bucket=%s' % bucket,
        '--sha1_file',
        tp(sha1_file),
    ],
        env=google_env())

```

### download_gn

从Google存储中下载`gn`。

```py
def download_gn():
    download_from_google_storage('gn', 'chromium-gn')
```

### download_clang_format

从Google存储中下载`clang-format`。

```py
def download_clang_format():
    download_from_google_storage('clang-format', 'chromium-clang-format')

```

### download_clang

通过调用clang更新脚本,来下载`clang`。

```py
def download_clang():
    run(['python',
         tp('v8/tools/clang/scripts/update.py'), '--if-needed'],
        env=google_env())

```

### maybe_download_sysroot

只在`linux` 中下载, `sysroot`

``` py
def maybe_download_sysroot():
    if sys.platform.startswith('linux'):
        run([
            'python',
            tp('v8/build/linux/sysroot_scripts/install-sysroot.py'),
            '--arch=amd64'
        ],
            env=google_env())

```