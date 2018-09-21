## rust.gni

``` bash
declare_args() {
  # Absolute path of rust build files.
  rust_build = "//build_extra/rust/"

  # treat the warnings in rust files as errors
  rust_treat_warnings_as_errors = true
}

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

template("rust_crate") {
  rustc_name = target_name + "_rustc"
  rustc_label = ":" + rustc_name
  config_name = target_name + "_config"

  # Convert all 'extern' and 'extern_version' items to a single format.
  extern_infos = []
  if (defined(invoker.extern)) {
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

template("rust_executable") {
  bin_name = target_name + "_bin"
  bin_label = ":" + bin_name

  rust_crate(bin_name) {
    crate_type = "bin"
    forward_variables_from(invoker, "*")
  }

  executable(target_name) {
    forward_variables_from(invoker, "*")

    if (defined(is_test) && is_test) {
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
      foreach(info, extern_version) {
        deps += [ info.label ]
      }
    }
  }
}

template("rust_test") {
  rust_executable(target_name) {
    forward_variables_from(invoker, "*")
    is_test = true
    testonly = true
  }
}

```