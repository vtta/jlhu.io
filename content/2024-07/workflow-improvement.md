+++
+++
### CI for building kernel and running tests

#### Toolchain

llvm+clang v18.1.7 [binary](https://github.com/llvm/llvm-project/releases/download/llvmorg-18.1.7/clang+llvm-18.1.7-x86_64-linux-gnu-ubuntu-18.04.tar.xz)

clearlinux <https://hub.docker.com/_/clearlinux>

#### Host kernel

The official clear linux kernel can be obtained according to these [steps](https://www.clearlinux.org/clear-linux-documentation/guides/kernel/kernel-development.html). The official has BTF debug info enabled, but not `/dev/mem`, `/dev/kmem` and `/proc/kcore`.

And we should also disable the mangle of pointer values in `dmesg` via `sudo sysctl -w kernel.kptr_restrict=0`

To enable debugging with `drgn`, we can enable [`/dev/mem`](https://cateee.net/lkddb/web-lkddb/DEVMEM.html), [`/dev/kmem`](https://cateee.net/lkddb/web-lkddb/DEVKMEM.html) and `/proc/kcore` in `.config`. [Link](https://unix.stackexchange.com/a/382924)

```
CONFIG_DEVKMEM
CONFIG_DEVMEM
CONFIG_STRICT_DEVMEM
CONFIG_PROC_KCORE
CONFIG_IKCONFIG
CONFIG_IKCONFIG_PROC
```

BUGS:

- `CONFIG_TRUSTED_KEYS_TPM=y` leads to compile time error because of the following code:

  ```c
   if (options->blobauth_len == 0) {
    unsigned char bool[3], *w = bool;
    /* tag 0 is emptyAuth */
    w = asn1_encode_boolean(w, w + sizeof(bool), true);
    if (WARN(IS_ERR(w), "BUG: Boolean failed to encode")) {
     ret = PTR_ERR(w);
     goto err;
    }
    work = asn1_encode_tag(work, end_work, 0, bool, w - bool);
   }
  ```

