# Mini A-IPC — Ubuntu Kernel Module Verification Guide

This guide starts from the beginning and checks whether your Ubuntu environment can build, load, and test the existing Mini A-IPC Linux kernel module.

> **Important:** Run these commands on Ubuntu (native installation or an Ubuntu VM), not in the WSL terminal unless you intentionally want to test the WSL kernel. Kernel headers must match the currently running kernel. Do not attempt to load the module if its `vermagic` does not match `uname -r`.

## Stage 1 — Check Ubuntu and the running kernel

Open a terminal and run:

```bash
cat /etc/os-release
uname -a
uname -r
uname -m
gcc --version
make --version
```

Check whether the matching kernel build directory exists:

```bash
ls -l /lib/modules/$(uname -r)/build
```

Expected: this path should exist and normally be a symbolic link to the matching kernel headers/build tree.

## Stage 2 — Install development dependencies

```bash
sudo apt update
sudo apt upgrade -y

sudo apt install -y \
  build-essential \
  linux-headers-$(uname -r) \
  git \
  make \
  gcc \
  libelf-dev \
  dwarves \
  bc \
  flex \
  bison \
  libssl-dev
```

Verify the headers again:

```bash
ls -l /lib/modules/$(uname -r)/build
```

If the package `linux-headers-$(uname -r)` cannot be found, do not substitute headers for a different kernel version. Record the output of `uname -r` and investigate the correct headers package for that distribution/kernel.

## Stage 3 — Locate and inspect the project

If the repository is in your home directory:

```bash
cd ~/Mini-A-IPC-main
pwd
ls
ls kernel/
cat kernel/Makefile
```

The kernel directory should contain at least:

```text
kernel/
├── Makefile
└── aipc_module.c
```

The usual external-module Makefile structure is:

```makefile
obj-m += aipc_module.o

all:
	$(MAKE) -C /lib/modules/$(shell uname -r)/build M=$(CURDIR) modules

clean:
	$(MAKE) -C /lib/modules/$(shell uname -r)/build M=$(CURDIR) clean
```

Compare this with your existing `kernel/Makefile`. Do not overwrite it blindly; preserve any project-specific targets.

## Stage 4 — Build the kernel module

From the project root:

```bash
cd ~/Mini-A-IPC-main
make kernel-module
```

If the root Makefile does not provide a working `kernel-module` target, build directly:

```bash
make -C /lib/modules/$(uname -r)/build \
  M="$PWD/kernel" \
  modules
```

Check that the module was generated:

```bash
ls -lh kernel/aipc_module.ko
```

If the build fails, save the complete error output. Do not bypass `MODPOST` or unresolved-symbol errors with warning-only options.

## Stage 5 — Verify kernel-version compatibility

Run:

```bash
modinfo -F vermagic kernel/aipc_module.ko
uname -r
```

The first version string in `vermagic` must match the running kernel release from `uname -r`. If they differ, stop. Do not run `insmod`.

Also inspect the module metadata:

```bash
modinfo kernel/aipc_module.ko
```

## Stage 6 — Load the module and verify the character device

Only after the build succeeds and the version check matches:

```bash
sudo insmod kernel/aipc_module.ko
```

Check that the module is loaded:

```bash
lsmod | grep aipc_module
```

Check that the device exists:

```bash
ls -l /dev/aipc
```

Inspect recent kernel messages:

```bash
sudo dmesg | tail -n 40
```

If `insmod` fails, capture the exact error and the recent `dmesg` output. Do not repeatedly retry without understanding the failure.

## Stage 7 — Inspect and run the user-space client

Inspect the available client usage and project instructions:

```bash
cd ~/Mini-A-IPC-main
./aipc-client
cat docs/kernel-module.md
```

Use the commands documented by the repository to test the device. Verify at minimum:

- A write and a read transfer the expected bytes.
- Received data matches sent data.
- An independent producer and consumer process can communicate, if the implementation and test client support this.
- Blocking reads/writes behave as documented.
- Nonblocking operations return the expected result when the device cannot proceed.
- `poll()` reports readiness as documented.

**Important:** A process writing data and then reading its own data back is only a basic device round-trip test. It does not by itself prove independent producer-consumer communication.

## Stage 8 — Run the kernel tests

Inspect available scripts and Makefile targets:

```bash
find tests/kernel -maxdepth 1 -type f -print
grep -nE 'kernel-test|test-device|test-load|kernel' Makefile
```

If the repository provides a `kernel-test` target, run:

```bash
make kernel-test
```

Otherwise, read each script before running it. For example:

```bash
sed -n '1,240p' tests/kernel/test_device.sh
sed -n '1,240p' tests/kernel/test_load_unload.sh
```

Run only the scripts that match the current device interface and follow their documented privilege requirements.

## Stage 9 — Unload the module and verify cleanup

After testing:

```bash
sudo rmmod aipc_module
```

Verify that the module is no longer loaded:

```bash
lsmod | grep aipc_module
```

Verify device cleanup:

```bash
ls -l /dev/aipc
```

The module should be absent from `lsmod`, and `/dev/aipc` should be removed if the driver creates and destroys the device node through the expected mechanism.

## Stage 10 — Re-run the user-space project tests

From the repository root:

```bash
cd ~/Mini-A-IPC-main
make clean
make
make test
```

Record the full output and the number of passing and failing tests.

## Final verification checklist

- [ ] Ubuntu version and running kernel recorded.
- [ ] Matching kernel headers/build directory exists.
- [ ] User-space project builds.
- [ ] Kernel module builds without errors.
- [ ] Module `vermagic` matches `uname -r`.
- [ ] Module loads successfully.
- [ ] `/dev/aipc` is created.
- [ ] Kernel messages show no relevant errors or warnings.
- [ ] Device read/write and data-integrity tests pass.
- [ ] Independent producer-consumer communication is tested, if supported.
- [ ] Blocking, nonblocking, and `poll()` behavior is tested, if implemented.
- [ ] Kernel tests pass.
- [ ] Module unloads cleanly and device resources are released.
- [ ] Existing user-space tests still pass.
- [ ] Test outputs and environment details are saved for the project report.

## Troubleshooting notes

### Missing kernel build directory

If this fails:

```bash
ls -l /lib/modules/$(uname -r)/build
```

install the headers package matching the running kernel:

```bash
sudo apt install linux-headers-$(uname -r)
```

If that exact package is unavailable, do not use headers for a different kernel version. Identify the distribution's appropriate kernel/header package first.

### `vermagic` mismatch

If the module's `vermagic` does not begin with the exact output of `uname -r`, rebuild it against the matching build tree. Do not attempt to load it.

### `insmod` fails

Capture:

```bash
sudo dmesg | tail -n 80
modinfo kernel/aipc_module.ko
uname -r
```

A successful compile does not guarantee that the module can load or that its runtime behavior is correct.

## Completion criterion

The kernel portion should be considered verified only after the module builds against the running kernel, passes the compatibility check, loads, creates and operates its device, passes applicable functional tests, and unloads cleanly. A successful compile alone is not sufficient evidence that the kernel module works.
