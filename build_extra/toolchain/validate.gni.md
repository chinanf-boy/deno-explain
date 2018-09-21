## validate.gni

``` bash
# Copyright 2018 the Deno authors. All rights reserved. MIT license.

import("//build/toolchain/cc_wrapper.gni")

# Verify that the cc_wrapper/toolchain combo correctly set up on Windows.
if (is_win && cc_wrapper != "") {
  cc_wrapper_toolchain = "//build_extra/toolchain/win:win_clang_$target_cpu"
  toolchain_supports_cc_wrapper = cc_wrapper_toolchain == default_toolchain &&
                                  cc_wrapper_toolchain == host_toolchain

  if (!toolchain_supports_cc_wrapper) {
    # Using print instead of assert-with-message for readability of the output.
    print("The 'cc_wrapper' option isn't supported by the default Windows" +
          " toolchain. To make it work, add these gn arguments:")
    print("  custom_toolchain=\"$cc_wrapper_toolchain\"")
    print("  host_toolchain=\"$cc_wrapper_toolchain\"")
    assert(toolchain_supports_cc_wrapper)
  }
}

```