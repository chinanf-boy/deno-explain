## deno 的 gn文件说明

以下文件的根目录是`deno/`

一些开始的说明

0. [gn-zh](https://github.com/chinanf-boy/gn-zh) gn 存储库的中文翻译: 最好先看看入门指南, 预预热
1. `//` 是 根目录的意思
2. `//build/config/BUILDCONFIG.gn` 这是一个快捷方式/{软连接} 到 v8的构建配置

> 所以并没有打算去读, 你就把它当成是, 万能语言编译工具链配置就好了 😊

3. `//testing/gtest:gtest`注意到 **:gtest**了没有, 这是一个语法糖,完整的意思是:

- 获取`//testing/gtest/BUILD.gn`文件中
- 声明为`gtest` ,所定义的内容

### .gn

`gn`生成的起始文件

``` bash
# GN元构建系统查找并使用,源项目的根目录的此文件来设置启动选项。 
# 有关此设置中的值的文档文件，在命令行运行“gn help dotfile”。

# 构建配置文件的位置。
buildconfig = "//build/config/BUILDCONFIG.gn"

# 这些是默认情况下检查的目标. 目标中的文件
# 匹配这些模式（请参阅格式的“gn help label_pattern”）
# 检查这个【】内的正确的依赖关系
# 当你运行“gn check”或“gn gen --check”时。
check_targets = []

default_args = {
  # 与deno无关的各种全局chrome args。
  proprietary_codecs = false
  safe_browsing_mode = 0
  toolkit_views = false
  use_aura = false
  use_dbus = false
  use_gio = false
  use_glib = false
  use_ozone = false
  use_udev = false

  # TODO（ry）我们可能希望在某些时候启用CFI. 目前禁用更为简单
  # .请参见http://clang.llvm.org/docs/ControlFlowIntegrity.html
  is_cfi = false

  is_component_build = false
  symbol_level = 1
  treat_warnings_as_errors = false
  rust_treat_warnings_as_errors = false

  # https://cs.chromium.org/chromium/src/docs/ccache_mac.md
  clang_use_chrome_plugins = false

  v8_deprecation_warnings = false
  v8_embedder_string = "-deno"
  v8_enable_gdbjit = false
  v8_enable_i18n_support = false
  v8_experimental_extra_library_files = []
  v8_extra_library_files = []
  v8_imminent_deprecation_warnings = false
  v8_monolithic = true
  v8_untrusted_code_mitigations = false
  v8_use_external_startup_data = false
  v8_use_snapshot = true
}
```

下一步则是同目录下的, [BUILD.gn](#BUILD.gn)

### //BUILD.gn

自定义构建流程

这是一个`gn`自身的语法(简单来讲)的文件

### 目录

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [import](#import)
- [group](#group)
  - [因为有点多, 挑重点讲`:deno`](#%E5%9B%A0%E4%B8%BA%E6%9C%89%E7%82%B9%E5%A4%9A-%E6%8C%91%E9%87%8D%E7%82%B9%E8%AE%B2deno)
- [config](#config)
- [main_extern](#main_extern)
- [rust_executable("deno")](#rust_executabledeno)
- [rust_executable("deno_ns")](#rust_executabledeno_ns)
- [static_library("libdeno")](#static_librarylibdeno)
- [v8_source_set("deno_base")](#v8_source_setdeno_base)
- [v8_source_set("deno_base_test")](#v8_source_setdeno_base_test)
- [v8_source_set("deno_bindings")](#v8_source_setdeno_bindings)
- [executable("snapshot_creator")](#executablesnapshot_creator)
- [run_node("gen_declarations")](#run_nodegen_declarations)
- [run_node("bundle")](#run_nodebundle)
- [action("bundle_hash_h")](#actionbundle_hash_h)
- [source_set("libdeno_nosnapshot")](#source_setlibdeno_nosnapshot)
- [ts_flatbuffer("msg_ts")](#ts_flatbuffermsg_ts)
- [rust_flatbuffer("msg_rs")](#rust_flatbuffermsg_rs)
- [create_snapshot("deno")](#create_snapshotdeno)
- [create_snapshot("libdeno_test")](#create_snapshotlibdeno_test)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

#### import

> `*.gni`与`*.gn`, 我猜啊, 为了与gn自身的流程`BUILD.gn`分别开来,适用于提供模版和参数的可配置文件

<!-- 
- [ ] [./build_extra/flatbuffers/flatbuffer.gni](./build_extra/flatbuffers/flatbuffer.gni.md)
- [ ] [./build_extra/flatbuffers/rust/rust_flatbuffer.gni](./build_extra/flatbuffers/rust/rust_flatbuffer.gni.md)
- [ ] [./build_extra/deno.gni](./build_extra/deno.gni.md)
- [ ] [./build_extra/rust/rust.gni](./build_extra/rust/rust.gni.md)
- [ ] [./build_extra/toolchain/validate.gni](./build_extra/toolchain/validate.gni.md) -->

``` bash
# Copyright 2018 the Deno authors. All rights reserved. MIT license.
import("//build/toolchain/cc_wrapper.gni")
import("//third_party/v8/gni/v8.gni")
import("//third_party/v8/snapshot_toolchain.gni")
import("//build_extra/flatbuffers/flatbuffer.gni")
import("//build_extra/flatbuffers/rust/rust_flatbuffer.gni")
import("//build_extra/deno.gni")
import("//build_extra/rust/rust.gni")
import("//build_extra/toolchain/validate.gni")
```

#### group

默认运行组块

##### 因为有点多, 挑重点讲`:deno`

``` bash
group("default") {
  testonly = true
  deps = [
    ":deno", # 只讲这里
    ":deno_ns",
    ":test_cc",
    ":test_rs",
  ]
}
```

1. `:deno` 是简写, 代表[rust_executable("deno")](#rust_executable\(\"deno\"\))

#### config

定义的配置, 稍后可快速添加

> 且带有一些平台条件

``` bash
config("deno_config") {
  include_dirs = [ "third_party/v8" ]  # This allows us to v8/src/base/ libraries.
  configs = [ "third_party/v8:external_config" ]
  if (is_debug) {
    defines = [ "DEBUG" ]
  }

  # Targets built with the `rust_executable()` template automatically pick up
  # these dependencies, but those built with `executable()` need them when they
  # have Rust inputs. Currently, there's only one such target, `test_cc`.
  if (is_mac) {
    libs = [ "resolv" ]
  }
  if (is_win) {
    libs = [ "userenv.lib" ]
  }
}

```

#### main_extern

主要的rust库

> `$rust_build` 从哪里来 ? [import("//build_extra/rust/rust.gni")](./build_extra/rust/rust.gni.md#declare_args)

``` bash
main_extern = [
  "$rust_build:hyper",
  "$rust_build:hyper_rustls",
  "$rust_build:futures",
  "$rust_build:libc",
  "$rust_build:log",
  "$rust_build:ring",
  "$rust_build:tempfile",
  "$rust_build:rand",
  "$rust_build:tokio",
  "$rust_build:url",
  "//build_extra/flatbuffers/rust:flatbuffers",
  ":msg_rs",
]
```

#### rust_executable("deno")

调用 *rust_executable* 模版函数

> `rust_executable`来自哪里? [import("//build_extra/rust/rust.gni") 的 `template("rust_executable")` 模版](./build_extra/rust/rust.gni.md#template\(\"rust_executable\"\))

``` bash
rust_executable("deno") {
  source_root = "src/main.rs"
  extern = main_extern
  deps = [
    ":libdeno",
  ]
}
```

- [:libdeno](#static_library\("libdeno"\))

#### rust_executable("deno_ns")

``` bash
# This target is for fast incremental development.
# When modifying the javascript runtime, this target will not go through the
# extra process of building a snapshot and instead load the bundle from disk.
# ns = no snapshot
rust_executable("deno_ns") {
  source_root = "src/main.rs"
  extern = main_extern
  deps = [
    ":libdeno_nosnapshot",
  ]
}

rust_test("test_rs") {
  source_root = "src/main.rs"
  extern = main_extern
  deps = [
    ":libdeno",
  ]
}

v8_executable("test_cc") {
  testonly = true
  sources = [
    "libdeno/test.cc",
  ]
  deps = [
    ":deno_base_test",
    "//testing/gtest:gtest",
  ]
  configs = [ ":deno_config" ]
}

```

#### static_library("libdeno")

``` bash
static_library("libdeno") {
  complete_static_lib = true
  sources = [
    "libdeno/from_snapshot.cc",
  ]
  inputs = [
    "$target_gen_dir/snapshot_deno.bin",
  ]
  deps = [
    ":create_snapshot_deno",
    ":deno_bindings",
  ]
  configs += [ ":deno_config" ]

  # from_snapshot.cc uses an assembly '.incbin' directive to embed the snapshot.
  # This causes trouble when using sccache: since the snapshot file is not
  # inlined by the c preprocessor, sccache doesn't take its contents into
  # consideration, leading to false-positive cache hits.
  # Maybe other caching tools have this issue too, but ccache is unaffected.
  # Therefore, if a cc_wrapper is used that isn't ccache, include a generated
  # header file that contains the the sha256 hash of the snapshot.
  if (cc_wrapper != "" && cc_wrapper != "ccache") {
    hash_h = "$target_gen_dir/bundle/hash.h"
    inputs += [ hash_h ]
    deps += [ ":bundle_hash_h" ]
    if (is_win) {
      cflags = [ "/FI" + rebase_path(hash_h, target_out_dir) ]
    } else {
      cflags = [
        "-include",
        rebase_path(hash_h, target_out_dir),
      ]
    }
  }
}

```

#### v8_source_set("deno_base")

``` bash
# Only functionality needed for libdeno_test and snapshot_creator
# In particular no flatbuffers, no assets, no rust, no msg handlers.
# Because snapshots are slow, it's important that snapshot_creator's
# dependencies are minimal.
v8_source_set("deno_base") {
  sources = [
    "libdeno/binding.cc",
    "libdeno/deno.h",
    "libdeno/file_util.cc",
    "libdeno/file_util.h",
    "libdeno/internal.h",
  ]
  public_deps = [
    "third_party/v8:v8_monolith",
  ]
  configs = [ ":deno_config" ]
}

```

#### v8_source_set("deno_base_test") 

``` bash
v8_source_set("deno_base_test") {
  testonly = true
  sources = [
    "libdeno/file_util_test.cc",
    "libdeno/from_snapshot.cc",
    "libdeno/libdeno_test.cc",
  ]
  inputs = [
    "$target_gen_dir/snapshot_libdeno_test.bin",
  ]
  deps = [
    ":create_snapshot_libdeno_test",
    ":deno_base",
    "//testing/gtest:gtest",
  ]
  defines = [ "LIBDENO_TEST" ]
  configs = [ ":deno_config" ]
}
```

#### v8_source_set("deno_bindings")

``` bash
v8_source_set("deno_bindings") {
  deps = [
    ":deno_base",
  ]
  configs = [ ":deno_config" ]
}
```

#### executable("snapshot_creator")

``` bash
executable("snapshot_creator") {
  sources = [
    "libdeno/snapshot_creator.cc",
  ]
  deps = [
    ":deno_base",
  ]
  configs += [ ":deno_config" ]
}

# Generates type declarations for files that need to be included
```

#### run_node("gen_declarations")

``` bash
# in the runtime bundle
run_node("gen_declarations") {
  out_dir = target_gen_dir
  sources = [
    "js/assets.ts",
    "js/compiler.ts",
    "js/console.ts",
    "js/deno.ts",
    "js/dispatch.ts",
    "js/errors.ts",
    "js/fetch.ts",
    "js/global-eval.ts",
    "js/globals.ts",
    "js/mkdir.ts",
    "js/os.ts",
    "js/read_file.ts",
    "js/text_encoding.ts",
    "js/timers.ts",
    "js/tsconfig.generated.json",
    "js/types.ts",
    "js/util.ts",
    "js/v8_source_maps.ts",
  ]
  outputs = [
    "$out_dir/types/globals.d.ts",
  ]
  deps = [
    ":msg_ts",
  ]
  args = [
    "./node_modules/typescript/bin/tsc",
    "-p",
    rebase_path("js/tsconfig.generated.json", root_build_dir),
    "--baseUrl",
    rebase_path(root_build_dir, root_build_dir),
    "--outFile",
    rebase_path("$out_dir/types/globals.js", root_build_dir),
  ]
}
```

#### run_node("bundle")

``` bash
run_node("bundle") {
  out_dir = "$target_gen_dir/bundle/"
  sources = [
    "js/assets.ts",
    "js/compiler.ts",
    "js/console.ts",
    "js/dispatch.ts",
    "js/errors.ts",
    "js/fetch.ts",
    "js/fetch_types.d.ts",
    "js/globals.ts",
    "js/main.ts",
    "js/mkdir.ts",
    "js/os.ts",
    "js/plugins.d.ts",
    "js/read_file.ts",
    "js/text_encoding.ts",
    "js/timers.ts",
    "js/types.ts",
    "js/util.ts",
    "js/v8_source_maps.ts",
    "rollup.config.js",
    "src/msg.fbs",
    "tsconfig.json",
  ]
  outputs = [
    out_dir + "main.js",
    out_dir + "main.js.map",
  ]
  deps = [
    ":gen_declarations",
    ":msg_ts",
  ]
  args = [
    "./node_modules/rollup/bin/rollup",
    "-c",
    rebase_path("rollup.config.js", root_build_dir),
    "-i",
    rebase_path("js/main.ts", root_build_dir),
    "-o",
    rebase_path(out_dir + "main.js", root_build_dir),
    "--sourcemapFile",
    rebase_path("."),
    "--silent",
    "--environment",
    "BASEPATH:" + rebase_path(".", root_build_dir),
  ]
}
```

#### action("bundle_hash_h")

``` bash
action("bundle_hash_h") {
  script = "//tools/sha256sum.py"
  inputs = get_target_outputs(":bundle")
  outputs = [
    "$target_gen_dir/bundle/hash.h",
  ]
  deps = [
    ":bundle",
  ]
  args = [
    "--format",
    "__attribute__((__unused__)) static const int dummy_%s = 0;",
    "--outfile",
    rebase_path(outputs[0], root_build_dir),
  ]
  foreach(input, inputs) {
    args += [
      "--infile",
      rebase_path(input, root_build_dir),
    ]
  }
}
```

#### source_set("libdeno_nosnapshot")

``` bash
source_set("libdeno_nosnapshot") {
  bundle_outputs = get_target_outputs(":bundle")
  bundle_location = rebase_path(bundle_outputs[0], root_build_dir)
  bundle_map_location = rebase_path(bundle_outputs[1], root_build_dir)
  inputs = bundle_outputs
  sources = [
    "libdeno/from_filesystem.cc",
  ]
  deps = [
    ":bundle",
    ":deno_bindings",
  ]
  configs += [ ":deno_config" ]
  defines = [
    "BUNDLE_LOCATION=\"$bundle_location\"",
    "BUNDLE_MAP_LOCATION=\"$bundle_map_location\"",
  ]
}
```

#### ts_flatbuffer("msg_ts")

``` bash
ts_flatbuffer("msg_ts") {
  sources = [
    "src/msg.fbs",
  ]
}
```

#### rust_flatbuffer("msg_rs")

``` bash
rust_flatbuffer("msg_rs") {
  sources = [
    "src/msg.fbs",
  ]
}

```

#### create_snapshot("deno")

``` bash
# Generates $target_gen_dir/snapshot_deno.bin
create_snapshot("deno") {
  js = "$target_gen_dir/bundle/main.js"
  source_map = "$target_gen_dir/bundle/main.js.map"
  deps = [
    ":bundle",
  ]
}

```

#### create_snapshot("libdeno_test")

``` bash
# Generates $target_gen_dir/snapshot_libdeno_test.bin
create_snapshot("libdeno_test") {
  testonly = true
  js = "libdeno/libdeno_test.js"
}

```