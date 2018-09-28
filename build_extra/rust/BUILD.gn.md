## //build_extra/rust/BUILD.gn

<!-- START doctoc -->
<!-- END doctoc -->

### import
``` bash
# Copyright 2018 the Deno authors. All rights reserved. MIT license.

# Dependencies between third party crates is mapped out here manually.  This is
# not so difficult and having it be tedious to add dependencies might help us
# avoid dependency hell later on. Always try to minimize dependencies.
# Versioning for third party rust crates is controlled in //Cargo.toml
# Use //tools/sync_third_party.py instead of running "cargo install".

import("rust.gni")

crates = "//third_party/rust_crates"
registry_github = "$crates/registry/src/github.com-1ecc6299db9ec823/"

rust_crate("libc") {
  source_root = "$registry_github/libc-0.2.42/src/lib.rs"
  features = [ "use_std" ]
}

rust_crate("url") {
  source_root = "$registry_github/url-1.7.1/src/lib.rs"
  extern = [
    ":matches",
    ":idna",
    ":percent_encoding",
  ]
}

rust_crate("percent_encoding") {
  source_root = "$registry_github/percent-encoding-1.0.1/lib.rs"
  args = [
    # TODO: Suppress some warnings at this moment
    # This should be removed when it's fixed in servo/rust-url repository
    # https://github.com/servo/rust-url/issues/455
    "-Aunused-imports",
    "-Adeprecated",
  ]
}

rust_crate("matches") {
  source_root = "$registry_github/matches-0.1.7/lib.rs"
}

rust_crate("idna") {
  source_root = "$registry_github/idna-0.1.5/src/lib.rs"
  extern = [
    ":matches",
    ":unicode_bidi",
    ":unicode_normalization",
  ]
}

rust_crate("unicode_bidi") {
  source_root = "$registry_github/unicode-bidi-0.3.4/src/lib.rs"
  extern = [ ":matches" ]
}

rust_crate("unicode_normalization") {
  source_root = "$registry_github/unicode-normalization-0.1.7/src/lib.rs"
}

rust_crate("log") {
  source_root = "$registry_github/log-0.4.3/src/lib.rs"
  extern = [ ":cfg_if" ]
}

rust_crate("cfg_if") {
  source_root = "$registry_github/cfg-if-0.1.4/src/lib.rs"
}

rust_crate("tempfile") {
  source_root = "$registry_github/tempfile-3.0.2/src/lib.rs"
  extern = [
    ":libc",
    ":rand",
    ":remove_dir_all",
    ":winapi",
  ]
}

rust_crate("rand") {
  source_root = "$registry_github/rand-0.4.2/src/lib.rs"
  features = [
    "std",
    "alloc",
  ]
  extern = [
    ":libc",
    ":winapi",
  ]
  if (is_mac) {
    libs = [ "Security.framework" ]
  }
}

rust_crate("remove_dir_all") {
  source_root = "$registry_github/remove_dir_all-0.5.1/src/lib.rs"
  extern = [ ":winapi" ]
}

rust_crate("winapi") {
  source_root = "$registry_github/winapi-0.3.5/src/lib.rs"
  features = [
    "basetsd",
    "cfg",
    "cfgmgr32",
    "errhandlingapi",
    "excpt",
    "fileapi",
    "guiddef",
    "handleapi",
    "inaddr",
    "in6addr",
    "ktmtypes",
    "libloaderapi",
    "lsalookup",
    "minwinbase",
    "minwindef",
    "mstcpip",
    "ntdef",
    "ntsecapi",
    "ntstatus",
    "processthreadsapi",
    "profileapi",
    "qos",
    "rpcndr",
    "sspi",
    "std",
    "subauth",
    "sysinfoapi",
    "timezoneapi",
    "vadefs",
    "vcruntime",
    "winbase",
    "wincred",
    "windef",
    "winerror",
    "winnt",
    "winreg",
    "winsock2",
    "ws2def",
    "ws2ipdef",
    "ws2tcpip",
    "wtypes",
    "wtypesbase",
  ]
}

# Old version of the 'winapi' crate, required by 'mio', 'miow', and 'iovec'.
# This exceptional! Generally we don't allow multiple versions of a crate.
# TODO: Remove this dependency. https://github.com/denoland/deno/issues/484
rust_crate("winapi-0.2") {
  crate_name = "winapi"
  crate_version = "0.2"
  source_root = "$registry_github/winapi-0.2.8/src/lib.rs"
}

# TODO: Remove this crate together with crate 'winapi-0.2'.
rust_crate("kernel32") {
  source_root = "$registry_github/kernel32-sys-0.2.2/src/lib.rs"
  extern_version = [
    {
      label = ":winapi-0.2"
      crate_name = "winapi"
      crate_version = "0.2"
    },
  ]
}

# TODO: Remove this crate together with crate 'winapi-0.2'.
rust_crate("ws2_32") {
  source_root = "$registry_github/ws2_32-sys-0.2.1/src/lib.rs"
  extern_version = [
    {
      label = ":winapi-0.2"
      crate_name = "winapi"
      crate_version = "0.2"
    },
  ]
}

rust_crate("futures") {
  source_root = "$registry_github/futures-0.1.23/src/lib.rs"
  features = [
    "use_std",
    "with-deprecated",
  ]
}

rust_crate("mio") {
  source_root = "$registry_github/mio-0.6.15/src/lib.rs"
  features = [
    "default",
    "with-deprecated",
  ]
  extern = [
    ":iovec",
    ":kernel32",
    ":lazycell",
    ":libc",
    ":log",
    ":miow",
    ":net2",
    ":slab",
  ]

  # TODO: Upgrade to a current version of the 'winapi' crate.
  # See https://github.com/denoland/deno/issues/484.
  extern_version = [
    {
      label = ":winapi-0.2"
      crate_name = "winapi"
      crate_version = "0.2"
    },
  ]
}

rust_crate("miow") {
  source_root = "$registry_github/miow-0.2.1/src/lib.rs"
  extern = [
    ":kernel32",
    ":net2",
    ":ws2_32",
  ]

  # TODO: Upgrade to a current version of the 'winapi' crate.
  # See https://github.com/denoland/deno/issues/484.
  extern_version = [
    {
      label = ":winapi-0.2"
      crate_name = "winapi"
      crate_version = "0.2"
    },
  ]
}

rust_crate("iovec") {
  source_root = "$registry_github/iovec-0.1.2/src/lib.rs"
  extern = [ ":libc" ]

  # TODO: Upgrade to a current version of the 'winapi' crate.
  # See https://github.com/denoland/deno/issues/484.
  extern_version = [
    {
      label = ":winapi-0.2"
      crate_name = "winapi"
      crate_version = "0.2"
    },
  ]
}

rust_crate("lazycell") {
  source_root = "$registry_github/lazycell-0.6.0/src/lib.rs"
  args = [
    # TODO Remove these:
    "-Aunused_unsafe",
    "-Aunused_mut",
  ]
}

rust_crate("net2") {
  source_root = "$registry_github/net2-0.2.33/src/lib.rs"
  features = [
    "default",
    "duration",
  ]
  extern = [
    ":cfg_if",
    ":libc",
    ":winapi",
  ]
}

rust_crate("slab") {
  source_root = "$registry_github/slab-0.4.1/src/lib.rs"
}

rust_crate("bytes") {
  source_root = "$registry_github/bytes-0.4.9/src/lib.rs"
  extern = [
    ":byteorder",
    ":iovec",
  ]
}

rust_crate("byteorder") {
  source_root = "$registry_github/byteorder-1.2.4/src/lib.rs"
}

rust_crate("crossbeam_deque") {
  source_root = "$registry_github/crossbeam-deque-0.3.1/src/lib.rs"
  features = [ "use_std" ]
  extern = [
    ":crossbeam_epoch",
    ":crossbeam_utils",
  ]
}

rust_crate("crossbeam_epoch") {
  source_root = "$registry_github/crossbeam-epoch-0.4.3/src/lib.rs"
  features = [
    "use_std",
    "lazy_static",
    "default",
    "crossbeam-utils",
  ]
  extern = [
    ":arrayvec",
    ":cfg_if",
    ":crossbeam_utils",
    ":lazy_static",
    ":memoffset",
    ":scopeguard",
  ]
}

rust_crate("crossbeam_utils") {
  source_root = "$registry_github/crossbeam-utils-0.3.2/src/lib.rs"
  features = [
    "use_std",
    "default",
  ]
  extern = [ ":cfg_if" ]
}

rust_crate("arrayvec") {
  source_root = "$registry_github/arrayvec-0.4.7/src/lib.rs"
  extern = [ ":nodrop" ]
}

rust_crate("nodrop") {
  source_root = "$registry_github/nodrop-0.1.12/src/lib.rs"
}

rust_crate("lazy_static") {
  source_root = "$registry_github/lazy_static-1.1.0/src/lib.rs"
  args = [
    "--cfg",
    "lazy_static_inline_impl",
  ]
}

rust_crate("memoffset") {
  source_root = "$registry_github/memoffset-0.2.1/src/lib.rs"
}

rust_crate("scopeguard") {
  source_root = "$registry_github/scopeguard-0.3.3/src/lib.rs"
  features = [ "use_std" ]
}

rust_crate("num_cpus") {
  source_root = "$registry_github/num_cpus-1.8.0/src/lib.rs"
  extern = [ ":libc" ]
}

rust_crate("hyper") {
  source_root = "$registry_github/hyper-0.12.8/src/lib.rs"
  features = [ "runtime" ]
  extern = [
    ":bytes",
    ":futures",
    ":futures_cpupool",
    ":h2",
    ":http",
    ":httparse",
    ":iovec",
    ":itoa",
    ":log",
    ":net2",
    ":time",
    ":tokio",
    ":tokio_executor",
    ":tokio_io",
    ":tokio_reactor",
    ":tokio_tcp",
    ":tokio_timer",
    ":want",
  ]
  args = [
    # TODO Remove these.
    "-Adeprecated",
    "-Aunused_imports",
  ]
}

rust_crate("tokio_core") {
  source_root = "$registry_github/tokio-core-0.1.17/src/lib.rs"
  extern = [
    ":mio",
    ":futures",
    ":tokio",
    ":tokio_executor",
    ":tokio_reactor",
    ":tokio_timer",
    ":tokio_io",
    ":log",
    ":iovec",
    ":bytes",
    ":scoped_tls",
  ]
}

rust_crate("h2") {
  source_root = "$registry_github/h2-0.1.12/src/lib.rs"
  extern = [
    ":byteorder",
    ":bytes",
    ":fnv",
    ":futures",
    ":http",
    ":indexmap",
    ":log",
    ":slab",
    ":string",
    ":tokio_io",
  ]
}

rust_crate("http") {
  source_root = "$registry_github/http-0.1.10/src/lib.rs"
  extern = [
    ":bytes",
    ":fnv",
    ":itoa",
  ]
}

rust_crate("httparse") {
  source_root = "$registry_github/httparse-1.3.2/src/lib.rs"
}

rust_crate("fnv") {
  source_root = "$registry_github/fnv-1.0.6/lib.rs"
}

rust_crate("futures_cpupool") {
  source_root = "$registry_github/futures-cpupool-0.1.8/src/lib.rs"
  extern = [
    ":futures",
    ":num_cpus",
  ]
}

rust_crate("indexmap") {
  source_root = "$registry_github/indexmap-1.0.1/src/lib.rs"
}

rust_crate("itoa") {
  source_root = "$registry_github/itoa-0.4.2/src/lib.rs"
  features = [ "std" ]
}

rust_crate("string") {
  source_root = "$registry_github/string-0.1.1/src/lib.rs"
}

rust_crate("time") {
  source_root = "$registry_github/time-0.1.40/src/lib.rs"
  extern = [
    ":libc",
    ":winapi",
  ]
}

rust_crate("try_lock") {
  source_root = "$registry_github/try-lock-0.2.2/src/lib.rs"
}

rust_crate("want") {
  source_root = "$registry_github/want-0.0.6/src/lib.rs"
  extern = [
    ":futures",
    ":try_lock",
    ":log",
  ]
}

rust_crate("tokio") {
  source_root = "$registry_github/tokio-0.1.7/src/lib.rs"
  extern = [
    ":futures",
    ":mio",
    ":tokio_executor",
    ":tokio_fs",
    ":tokio_io",
    ":tokio_reactor",
    ":tokio_tcp",
    ":tokio_threadpool",
    ":tokio_timer",
    ":tokio_udp",
  ]
}

rust_crate("tokio_executor") {
  source_root = "$registry_github/tokio-executor-0.1.3/src/lib.rs"
  extern = [ ":futures" ]
}

rust_crate("tokio_fs") {
  source_root = "$registry_github/tokio-fs-0.1.3/src/lib.rs"
  extern = [
    ":futures",
    ":tokio_io",
    ":tokio_threadpool",
  ]
}

rust_crate("tokio_io") {
  source_root = "$registry_github/tokio-io-0.1.7/src/lib.rs"
  extern = [
    ":bytes",
    ":futures",
    ":log",
  ]
}

rust_crate("tokio_timer") {
  source_root = "$registry_github/tokio-timer-0.2.5/src/lib.rs"
  extern = [
    ":futures",
    ":tokio_executor",
  ]
}

rust_crate("tokio_udp") {
  source_root = "$registry_github/tokio-udp-0.1.1/src/lib.rs"
  extern = [
    ":bytes",
    ":futures",
    ":log",
    ":mio",
    ":tokio_codec",
    ":tokio_io",
    ":tokio_reactor",
  ]
}

rust_crate("tokio_codec") {
  source_root = "$registry_github/tokio-codec-0.1.0/src/lib.rs"
  extern = [
    ":bytes",
    ":futures",
    ":tokio_io",
  ]
}

rust_crate("tokio_reactor") {
  source_root = "$registry_github/tokio-reactor-0.1.3/src/lib.rs"
  extern = [
    ":futures",
    ":log",
    ":mio",
    ":slab",
    ":tokio_executor",
    ":tokio_io",
  ]
}

rust_crate("tokio_tcp") {
  source_root = "$registry_github/tokio-tcp-0.1.1/src/lib.rs"
  extern = [
    ":bytes",
    ":futures",
    ":iovec",
    ":mio",
    ":tokio_io",
    ":tokio_reactor",
  ]
}

rust_crate("tokio_threadpool") {
  source_root = "$registry_github/tokio-threadpool-0.1.5/src/lib.rs"
  extern = [
    ":crossbeam_deque",
    ":crossbeam_utils",
    ":futures",
    ":log",
    ":num_cpus",
    ":rand",
    ":tokio_executor",
  ]
}

rust_crate("hyper_rustls") {
  source_root = "$registry_github/hyper-rustls-0.14.0/src/lib.rs"
  extern = [
    ":ct_logs",
    ":futures",
    ":http",
    ":hyper",
    ":rustls",
    ":tokio_core",
    ":tokio_io",
    ":tokio_rustls",
    ":tokio_tcp",
    ":webpki",
    ":webpki_roots",
  ]
}

ring_root = "$registry_github/ring-0.13.2/"

component("ring_primitives") {
  sources = [
    "$ring_root/crypto/constant_time_test.c",
    "$ring_root/crypto/cpu-aarch64-linux.c",
    "$ring_root/crypto/cpu-arm-linux.c",
    "$ring_root/crypto/cpu-arm.c",
    "$ring_root/crypto/cpu-intel.c",
    "$ring_root/crypto/crypto.c",
    "$ring_root/crypto/fipsmodule/aes/aes.c",
    "$ring_root/crypto/fipsmodule/aes/internal.h",
    "$ring_root/crypto/fipsmodule/bn/exponentiation.c",
    "$ring_root/crypto/fipsmodule/bn/generic.c",
    "$ring_root/crypto/fipsmodule/bn/internal.h",
    "$ring_root/crypto/fipsmodule/bn/montgomery.c",
    "$ring_root/crypto/fipsmodule/bn/montgomery_inv.c",
    "$ring_root/crypto/fipsmodule/bn/shift.c",
    "$ring_root/crypto/fipsmodule/cipher/e_aes.c",
    "$ring_root/crypto/fipsmodule/cipher/internal.h",
    "$ring_root/crypto/fipsmodule/ec",
    "$ring_root/crypto/fipsmodule/ec/ecp_nistz.c",
    "$ring_root/crypto/fipsmodule/ec/ecp_nistz.h",
    "$ring_root/crypto/fipsmodule/ec/ecp_nistz256.c",
    "$ring_root/crypto/fipsmodule/ec/ecp_nistz256.h",
    "$ring_root/crypto/fipsmodule/ec/ecp_nistz384.h",
    "$ring_root/crypto/fipsmodule/ec/gfp_p256.c",
    "$ring_root/crypto/fipsmodule/ec/gfp_p384.c",
    "$ring_root/crypto/fipsmodule/modes/gcm.c",
    "$ring_root/crypto/fipsmodule/modes/internal.h",
    "$ring_root/crypto/internal.h",
    "$ring_root/crypto/limbs/limbs.c",
    "$ring_root/crypto/limbs/limbs.h",
    "$ring_root/crypto/mem.c",
    "$ring_root/include/GFp/aes.h",
    "$ring_root/include/GFp/arm_arch.h",
    "$ring_root/include/GFp/base.h",
    "$ring_root/include/GFp/cpu.h",
    "$ring_root/include/GFp/mem.h",
    "$ring_root/include/GFp/type_check.h",
    "$ring_root/third_party/fiat/curve25519.c",
    "$ring_root/third_party/fiat/curve25519_tables.h",
    "$ring_root/third_party/fiat/internal.h",

    #"$ring_root/crypto/fipsmodule/modes/polyval.c",
  ]
  if (is_mac) {
    sources += [
      "$ring_root/pregenerated/aes-586-macosx.S",
      "$ring_root/pregenerated/aes-x86_64-macosx.S",
      "$ring_root/pregenerated/aesni-gcm-x86_64-macosx.S",
      "$ring_root/pregenerated/aesni-x86-macosx.S",
      "$ring_root/pregenerated/aesni-x86_64-macosx.S",
      "$ring_root/pregenerated/chacha-x86-macosx.S",
      "$ring_root/pregenerated/chacha-x86_64-macosx.S",
      "$ring_root/pregenerated/ecp_nistz256-x86-macosx.S",
      "$ring_root/pregenerated/ghash-x86-macosx.S",
      "$ring_root/pregenerated/ghash-x86_64-macosx.S",
      "$ring_root/pregenerated/p256-x86_64-asm-macosx.S",
      "$ring_root/pregenerated/poly1305-x86-macosx.S",
      "$ring_root/pregenerated/poly1305-x86_64-macosx.S",
      "$ring_root/pregenerated/sha256-586-macosx.S",
      "$ring_root/pregenerated/sha256-x86_64-macosx.S",
      "$ring_root/pregenerated/sha512-586-macosx.S",
      "$ring_root/pregenerated/sha512-x86_64-macosx.S",
      "$ring_root/pregenerated/vpaes-x86-macosx.S",
      "$ring_root/pregenerated/vpaes-x86_64-macosx.S",
      "$ring_root/pregenerated/x86-mont-macosx.S",
      "$ring_root/pregenerated/x86_64-mont-macosx.S",
      "$ring_root/pregenerated/x86_64-mont5-macosx.S",
    ]
  }
  if (is_linux) {
    sources += [
      "$ring_root/pregenerated/aes-x86_64-elf.S",
      "$ring_root/pregenerated/aesni-gcm-x86_64-elf.S",
      "$ring_root/pregenerated/aesni-x86_64-elf.S",
      "$ring_root/pregenerated/aesv8-armx-linux64.S",
      "$ring_root/pregenerated/chacha-x86_64-elf.S",
      "$ring_root/pregenerated/ghash-x86_64-elf.S",
      "$ring_root/pregenerated/ghashv8-armx-linux64.S",
      "$ring_root/pregenerated/p256-x86_64-asm-elf.S",
      "$ring_root/pregenerated/poly1305-x86_64-elf.S",
      "$ring_root/pregenerated/sha256-x86_64-elf.S",
      "$ring_root/pregenerated/sha512-x86_64-elf.S",
      "$ring_root/pregenerated/vpaes-x86_64-elf.S",
      "$ring_root/pregenerated/x86_64-mont-elf.S",
      "$ring_root/pregenerated/x86_64-mont5-elf.S",
    ]
  }
  if (is_win) {
    libs = [
      "$ring_root/pregenerated/aes-x86_64-nasm.obj",
      "$ring_root/pregenerated/aesni-gcm-x86_64-nasm.obj",
      "$ring_root/pregenerated/aesni-x86_64-nasm.obj",
      "$ring_root/pregenerated/chacha-x86_64-nasm.obj",
      "$ring_root/pregenerated/ghash-x86_64-nasm.obj",
      "$ring_root/pregenerated/p256-x86_64-asm-nasm.obj",
      "$ring_root/pregenerated/poly1305-x86_64-nasm.obj",
      "$ring_root/pregenerated/sha256-x86_64-nasm.obj",
      "$ring_root/pregenerated/sha512-x86_64-nasm.obj",
      "$ring_root/pregenerated/vpaes-x86_64-nasm.obj",
      "$ring_root/pregenerated/x86_64-mont-nasm.obj",
      "$ring_root/pregenerated/x86_64-mont5-nasm.obj",
    ]
  }
  include_dirs = [ "$ring_root/include/" ]
}

rust_crate("ring") {
  source_root = "$ring_root/src/lib.rs"
  features = [
    "use_heap",
    "rsa_signing",
  ]
  extern = [
    ":libc",
    ":untrusted",
    ":lazy_static",
  ]
  deps = [
    ":ring_primitives",
  ]
}

rust_crate("rustls") {
  source_root = "$registry_github/rustls-0.13.1/src/lib.rs"
  extern = [
    ":untrusted",
    ":base64",
    ":log",
    ":ring",
    ":webpki",
    ":sct",
  ]
  args = [ "-Aunused_variables" ]  # TODO Remove.
}

rust_crate("ct_logs") {
  source_root = "$registry_github/ct-logs-0.4.0/src/lib.rs"
  extern = [ ":sct" ]
}

rust_crate("tokio_rustls") {
  source_root = "$registry_github/tokio-rustls-0.7.2/src/lib.rs"
  extern = [
    ":rustls",
    ":webpki",
    ":tokio",
  ]
  features = [
    "default",
    "tokio",
    "tokio-support",
  ]
  args = [ "-Adead_code" ]  # TODO Remove.
}

rust_crate("untrusted") {
  source_root = "$registry_github/untrusted-0.6.2/src/untrusted.rs"
  extern = []
}

rust_crate("webpki") {
  source_root = "$registry_github/webpki-0.18.1/src/webpki.rs"
  features = [
    "std",
    "trust_anchor_util",
  ]
  extern = [
    ":ring",
    ":untrusted",
  ]
}

rust_crate("webpki_roots") {
  source_root = "$registry_github/webpki-roots-0.15.0/src/lib.rs"
  extern = [
    ":webpki",
    ":untrusted",
  ]
}

rust_crate("sct") {
  source_root = "$registry_github/sct-0.4.0/src/lib.rs"
  extern = [
    ":ring",
    ":untrusted",
  ]
}

rust_crate("base64") {
  source_root = "$registry_github/base64-0.9.2/src/lib.rs"
  extern = [
    ":byteorder",
    ":safemem",
  ]
}

rust_crate("safemem") {
  source_root = "$registry_github/safemem-0.2.0/src/lib.rs"
}

rust_crate("scoped_tls") {
  source_root = "$registry_github/scoped-tls-0.1.2/src/lib.rs"
  extern = [
    ":ring",
    ":untrusted",
  ]
}

rust_crate("smallvec") {
  source_root = "$registry_github/smallvec-0.6.5/lib.rs"
  extern = [ ":unreachable" ]
  features = [ "std" ]
}

rust_crate("unreachable") {
  source_root = "$registry_github/unreachable-1.0.0/src/lib.rs"
  extern = [ ":void" ]
}

rust_crate("void") {
  source_root = "$registry_github/void-1.0.2/src/lib.rs"
  features = [ "default" ]
}

```