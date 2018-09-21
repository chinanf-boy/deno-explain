## deno.gni

``` bash
# Copyright 2018 the Deno authors. All rights reserved. MIT license.
import("//build/compiled_action.gni")

template("run_node") {
  action(target_name) {
    forward_variables_from(invoker, "*")
    script = "//tools/run_node.py"
  }
}

# Template to generate different V8 snapshots based on different runtime flags.
# Can be invoked with run_mksnapshot(<name>). The target will resolve to
# run_mksnapshot_<name>. If <name> is "default", no file suffixes will be used.
# Otherwise files are suffixed, e.g. embedded_<name>.cc and
# snapshot_blob_<name>.bin.
#
# The template exposes the variables:
#   args: additional flags for mksnapshots
#   embedded_suffix: a camel case suffix for method names in the embedded
#       snapshot.
template("create_snapshot") {
  name = target_name
  compiled_action("create_snapshot_" + name) {
    forward_variables_from(invoker,
                           [
                             "testonly",
                             "deps",
                           ])
    tool = ":snapshot_creator"
    visibility = [ ":*" ]  # Only targets in this file can depend on this.
    snapshot_out_bin = "$target_gen_dir/snapshot_$name.bin"
    inputs = [
      invoker.js,
    ]
    if (defined(invoker.source_map)) {
      inputs += [ invoker.source_map ]
    }
    outputs = [
      snapshot_out_bin,
    ]
    args = rebase_path(outputs, root_build_dir) +
           rebase_path(inputs, root_build_dir)

    # To debug snapshotting problems:
    #  args += ["--trace-serializer"]
  }
}

```