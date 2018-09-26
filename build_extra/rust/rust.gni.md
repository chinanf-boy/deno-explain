## rust.gni

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [declare_args](#declare_args)
- [变量](#%E5%8F%98%E9%87%8F)
- [1. template("run_rustc")](#1-templaterun_rustc)
- [2. template("rust_crate")](#2-templaterust_crate)
- [3. template("rust_executable")](#3-templaterust_executable)
  - [3.1 rust_crate(bin_name)](#31-rust_cratebin_name)
  - [3.2 executable(target_name)](#32-executabletarget_name)
- [4. template("rust_test")](#4-templaterust_test)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

### declare_args

自定义构建参数

``` bash
declare_args() {
  # rust 构建文件的绝对路径.
  rust_build = "//build_extra/rust/"

  # 看待 rust 文件中的警告⚠️,为错误❌
  rust_treat_warnings_as_errors = true
}

```

[gn 入门指南#添加新的构建参数](https://github.com/chinanf-boy/gn-zh/blob/master/docs/quick_start.zh.md#%E6%B7%BB%E5%8A%A0%E6%96%B0%E7%9A%84%E6%9E%84%E5%BB%BA%E5%8F%82%E6%95%B0)

### 变量

``` bash
if (is_win) {
  executable_suffix = ".exe"
} else {
  executable_suffix = ""
}

# To simplify transitive dependency management with gn, we build all rust
# crates into the same directory. We need to be careful not to have crates
# with the same name.
out_dir = "$root_out_dir/rust_crates"

# The official way of building Rust executables is to to let rustc do the
# linking. However, we'd prefer to leave it in the hands of gn/ninja:
#   * It allows us to use source sets.
#   * It allows us to use the bundled lld that Chromium and V8 use.
#   * We have more control over build flags.
#   * To sidestep rustc weirdness (e.g. on Windows, it always links with the
#     release C runtime library, even for debug builds).
#
# The `get_rust_ldflags` tool outputs the linker flags that are needed to
# successfully link rustc object code into an executable.
# We generate two sets of ldflags:
#   `rust_bin_ldflags`:  Used for rust_executable targets.
#   `rust_test_ldflags`: Used for rust_test targets; includes the test harness.
#
# The tool works by compiling and linking something with rustc, and analyzing
# the arguments it passes to the system linker. That's what dummy.rs is for.
dummy_rs_path = rebase_path("dummy.rs", root_build_dir)
rust_bin_ldflags =
    exec_script("get_rust_ldflags.py", [ dummy_rs_path ], "list lines")
rust_test_ldflags = exec_script("get_rust_ldflags.py",
                                [
                                  dummy_rs_path,
                                  "--test",
                                ],
                                "list lines")

```

### 1. template("run_rustc")

``` bash
template("run_rustc") {
  action(target_name) {
    assert(defined(invoker.source_root), "Must specify source_root")
    forward_variables_from(invoker,
                           [
                             "crate_name",
                             "crate_type",
                             "crate_version",
                             "deps",
                             "extern_infos",
                             "features",
                             "is_test",
                             "source_root",
                             "testonly",
                           ])
    if (!defined(crate_name)) {
      crate_name = target_name
    }
    if (!defined(is_test)) {
      is_test = false
    }

    sources = [
      source_root,
    ]
    script = "//tools/run_rustc.py"

    args = [
      rebase_path(source_root, root_build_dir),
      "--crate-name=$crate_name",
      "--crate-type=$crate_type",
    ]

    if (rust_treat_warnings_as_errors) {
      args += [ "-Dwarnings" ]
    }

    if (!is_win) {
      args += [ "--color=always" ]
    }

    if (!defined(crate_version)) {
      crate_name_and_version = crate_name
    } else {
      crate_name_and_version = "$crate_name-$crate_version"
    }

    if (crate_type == "bin") {
      output_file = "$out_dir/$crate_name_and_version.o"
      emit_type = "obj"
    } else if (crate_type == "rlib") {
      output_file = "$out_dir/lib$crate_name_and_version.rlib"
      emit_type = "link"
    }
    outputs = [
      output_file,
    ]
    output_file_rel = rebase_path(output_file, root_build_dir)
    args += [ "--emit=$emit_type=$output_file_rel" ]

    depfile = "$out_dir/$crate_name_and_version.d"
    args += [
      "--emit=dep-info=" + rebase_path(depfile, root_build_dir),

      # The following two args are used by run_rustc.py to fix
      # the depfile on the fly. They are not passed thru to rustc.
      "--depfile=" + rebase_path(depfile, root_build_dir),
      "--output_file=" + output_file_rel,

      # This is needed for transitive dependencies.
      "-L",
      "dependency=" + rebase_path(out_dir, root_build_dir),
    ]

    if (defined(crate_version)) {
      # Compute the sha256sum of the version number. See comments below.
      # Note that we do this only if there are multiple versions of this crate.
      hash = exec_script("//tools/sha256sum.py",
                         [
                           "--input",
                           crate_version,
                           "--format",
                           "%.8s",
                         ],
                         "trim string")

      args += [
        # In our build setup, all crates are built in the same directory. The
        # actual build outputs have unique names (e.g. 'foo-1.2.3.rlib'), but
        # rustc also creates many temp files (e.g. 'foo.crate.metadata.rcgu.bc',
        # 'foo.foo0.rcgu.o'), and with those files, name collisions do occur.
        # This causes random failures during parallel builds and on CI.
        #
        # These name conflicts can be avoided by setting `-C extra-filename=` to
        # some unique value. Unfortunately the version number as such can't be
        # used: everything after the first dot (.) is thrown away, so winapi-0.2
        # vs. winapi-0.3 would still have conflicts -- so we use a hash instead.
        "-C",
        "extra-filename=$hash",

        # Rustc blows up if a target (directly or indirectly) depends on two+
        # crates that have the same name *and* the same metadata. So we use the
        # hash to give 'metadata' a unique value too (any unique value will do).
        "-C",
        "metadata=$hash",
      ]
    }

    if (is_debug) {
      args += [ "-g" ]
    }

    if (is_official_build) {
      args += [ "-O" ]
    }

    if (is_test) {
      args += [ "--test" ]
    }

    if (defined(features)) {
      foreach(f, features) {
        args += [
          "--cfg",
          "feature=\"" + f + "\"",
        ]
      }
    }

    if (defined(invoker.args)) {
      args += invoker.args
    }

    if (!defined(deps)) {
      deps = []
    }

    # Build the list of '--extern' arguments from the 'extern_infos' array.
    foreach(info, extern_infos) {
      rlib = "$out_dir/lib${info.crate_name_and_version}.rlib"
      args += [
        "--extern",
        info.crate_name + "=" + rebase_path(rlib, root_build_dir),
      ]
      deps += [ info.label ]
    }
  }
}

```

### 2. template("rust_crate")

> **target_name** === `deno_bin`

``` bash
template("rust_crate") {
  rustc_name = target_name + "_rustc"
  rustc_label = ":" + rustc_name
  config_name = target_name + "_config"

  # 转换 每个'extern' 和 'extern_version' , 到一个 单 格式.
  extern_infos = []
  if (defined(invoker.extern)) { # invoker.extern 来自根目录 rust_executable("deno") 的调用参数
    foreach(label, invoker.extern) {
      extern_infos += [
        {
          label = label
          crate_name = get_label_info(label, "name")
          crate_name_and_version = crate_name
        },
      ]
    }
  }
  if (defined(invoker.extern_version)) {
    foreach(info, invoker.extern_version) {
      extern_infos += [
        {
          forward_variables_from(info,
                                 [
                                   "label",
                                   "crate_name",
                                   "crate_version",
                                 ])
          crate_name_and_version = "$crate_name-$crate_version"
        },
      ]
    }
  }

  forward_variables_from(invoker,
                         [
                           "crate_name",
                           "crate_type",
                         ])
  if (!defined(crate_name)) {
    crate_name = target_name
  }
  if (!defined(crate_type)) {
    crate_type = "rlib"
  }

  run_rustc(rustc_name) {
    forward_variables_from(invoker,
                           [
                             "args",
                             "crate_version",
                             "deps",
                             "features",
                             "is_test",
                             "source_root",
                             "testonly",
                           ])
  }

  crate_outputs = get_target_outputs(rustc_label)
  crate_obj = crate_outputs[0]

  config(config_name) {
    lib_dirs = []
    forward_variables_from(invoker, [ "libs" ])
    if (!defined(libs)) {
      libs = []
    }
    foreach(info, extern_infos) {
      rlib = "$out_dir/lib${info.crate_name_and_version}.rlib"
      libs += [ rlib ]
    }
    lib_dirs = [ out_dir ]
  }

  source_set(target_name) {
    forward_variables_from(invoker,
                           [
                             "deps",
                             "libs",
                             "testonly",
                           ])
    if (!defined(deps)) {
      deps = []
    }
    if (!defined(libs)) {
      libs = []
    }
    libs += [ crate_obj ]
    deps += [ rustc_label ]
    all_dependent_configs = [ ":" + config_name ]
  }
}

```

### 3. template("rust_executable") 

定义模版函数,

> [gn template 用法](https://github.com/chinanf-boy/gn-zh/blob/master/docs/language.zh.md#%E6%A8%A1%E6%9D%BF)

``` bash
template("rust_executable") {
  bin_name = target_name + "_bin" # 这里target_name, 就是 deno
  bin_label = ":" + bin_name

```

#### 3.1 rust_crate(bin_name)

> `bin_name` === `deno_bin`

``` bash
  rust_crate(bin_name) {
    crate_type = "bin"
    forward_variables_from(invoker, "*") # 这里,
    # 上层 rust_executable("deno") 调用 template("rust_executable") 模版,的所有变量
    # 都能给到`rust_crate`模版中的, invoker 使用
  }

```

- [forward_variables_from] : 复制来自不同范围的变量.

[另一个模版 `rust_crate(bin_name)`](#2-template\(\"rust_crate\"\))

#### 3.2 executable(target_name)

> gn内置类型函数`executable`:生成一个可执行文件. [更多](https://github.com/chinanf-boy/gn-zh/blob/master/docs/language.zh.md#%E7%9B%AE%E6%A0%87)

- [forward_variables_from] : 复制来自不同范围的变量.

[forward_variables_from]: https://github.com/chinanf-boy/gn-zh/blob/master/docs/reference.md#forward_variables_from

``` bash
  executable(target_name) {
    forward_variables_from(invoker, "*") # 复制invoker的所有变量到本地括号闭包内

    if (defined(is_test) && is_test) { # defined 是否有定义变量
      ldflags = rust_test_ldflags
    } else {
      ldflags = rust_bin_ldflags
    }

    if (!defined(deps)) {
      deps = []
    }

    deps += [ bin_label ]

    if (defined(extern)) {
      deps += extern
    }
    if (defined(extern_version)) {
      foreach(info, extern_version) {  # foreach 迭代一个列表
        deps += [ info.label ]
      }
    }
  }
}

```

### 4. template("rust_test")

``` bash
template("rust_test") {
  rust_executable(target_name) {
    forward_variables_from(invoker, "*")
    is_test = true
    testonly = true
  }
}

```