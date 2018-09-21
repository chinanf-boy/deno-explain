## rust_flatbuffer.gni

``` bash
import("//build_extra/rust/rust.gni")

# TODO(ry) "flatbuffer.gni" should be "flatbuffers.gni" we should be consistent
# in our pluralization.
import("//build_extra/flatbuffers/flatbuffer.gni")

template("rust_flatbuffer") {
  action_name = "${target_name}_gen"
  source_set_name = target_name
  compiled_action_foreach(action_name) {
    tool = "$flatbuffers_build_location:flatc"

    sources = invoker.sources
    deps = []
    out_dir = target_gen_dir

    outputs = [
      "$out_dir/{{source_name_part}}_generated.rs",
    ]

    args = [
      "--rust",
      "-o",
      rebase_path(out_dir, root_build_dir),
      "-I",
      rebase_path("//", root_build_dir),
    ]
    args += [ "{{source}}" ]

    # The deps may have steps that have to run before running flatc.
    if (defined(invoker.deps)) {
      deps += invoker.deps
    }
  }

  rust_crate(source_set_name) {
    sources = get_target_outputs(":$action_name")
    source_root = sources[0]
    deps = [
      ":$action_name",
    ]
    extern = [ "//build_extra/flatbuffers/rust:flatbuffers" ]
  }

}
```