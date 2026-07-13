# Kernel Source Fixes for Realme RMX1941 (Lizard)

Custom kernel build: **v4.9.290-lizard+** (base: frostg-012 kernel_realme_RMX1941-lizard)

## Changes Summary

### 1. `fs/crypto/policy.c` — fscrypt boot fix (ROOT CAUSE)

**Problem:** OPPO init triggers `reboot("recovery")` during boot because `fscrypt_ioctl_set_policy`
returns `-EEXIST` with "Policy inconsistent with encryption context". The stock kernel (4.9.190) has
different fscrypt logic than the upstream 4.9.290 code in this source tree.

**Fix:** Reverted `fscrypt_ioctl_set_policy()` and `fscrypt_ioctl_get_policy()` to match the stock
4.9.190 kernel behavior:

- Removed duplicate `copy_from_user()` call (line 75-76 in broken code)
- Restored 3-branch logic in `set_policy`:
  1. `-ENODATA` → create new context
  2. Context matches policy → `ret = 0` (success, no-op)
  3. Different policy → `ret = -EEXIST`
- Added `AES_256_XTS` mode override in `get_policy` for Android FBE compliance

This is a **1:1 restore of stock kernel logic**, not new code.

### 2. `kernel/sched/wait.c` + `include/linux/wait.h` — `wake_up_pollfree` backport

Added `__wake_up_pollfree()` function using `EPOLLHUP | POLLFREE` flags.
Matches upstream kernel implementation. Used by `eventpoll.c` for cleanup on file descriptor close.

### 3. Connectivity drivers: built-in → module (WiFi/BT fix)

**Problem:** All connectivity drivers (`wmt_drv`, `bt_drv`, `gps_drv`, `fmradio_drv`, `wmt_chrdev_wifi`, `wlan_drv_gen4m`) were compiled as built-in (`obj-y`) in the kernel. This conflicts with the vendor `.ko` modules in `/vendor/lib/modules/` which provide the actual working WiFi/BT/GPS/FM drivers. On stock kernel (4.9.190+), ALL connectivity drivers are loaded as modules from vendor partition.

With drivers built-in:
- Vendor `wmt_drv.ko` fails with "duplicate symbol board_sdio_ctrl (owned by kernel)"
- Vendor `wmt_drv.ko` fails chardev registration (`/dev/stpwmt` already exists)
- Result: WiFi, Bluetooth, GPS, FM Radio all non-functional

**Fix:** Changed 6 Makefiles from `obj-y` to `obj-m` so connectivity drivers build as loadable modules:

| File | Change |
|------|--------|
| `drivers/misc/mediatek/connectivity/common/Makefile` | `obj-y` → `obj-m` for `wmt_drv` (else branch) |
| `drivers/misc/mediatek/connectivity/bt/mt66xx/Makefile` | `obj-y` → `obj-m` for `bt_drv` |
| `drivers/misc/mediatek/connectivity/gps/Makefile` | `obj-y` → `obj-m` for `gps_drv` |
| `drivers/misc/mediatek/connectivity/fmradio/Makefile` | `obj-y` → `obj-m` for `fmradio_drv` |
| `drivers/misc/mediatek/connectivity/wlan/adaptor/Makefile` | `obj-y` → `obj-m` for `wmt_chrdev_wifi` (else branch) |
| `drivers/misc/mediatek/connectivity/wlan/core/gen4m/Makefile` | `obj-y` → `obj-m` for `wlan_drv_gen4m` (else branch) |

### 4. Makefile `-I` include path fixes (multiple files)

Added missing `-I<path>` include flags to ~20 Makefiles to resolve header-not-found errors
during compilation. These only add include search paths — no code logic changes.

Affected directories: devfreq, aee/mrdump, ppm_v3, spm, ccu, cmdq, debug_latch, eccci,
emi, ext_disp, gud, leds, m4u, perf, smi, sspm, timesync, video, mmc, battery,
oppo_criticallog, oppo_hypnus, ion, wdt.

### 5. Config changes

- `arch/arm64/configs/RMX1941-stock_defconfig`: Added `CONFIG_MODULE_FORCE_LOAD=y` and `CONFIG_MODULE_FORCE_UNLOAD=y`
- `.config`: `CONFIG_DEBUG_INFO=n` (strips debug symbols, reduces vmlinux from 225MB to 38MB)
- `.config`: `CONFIG_DEFAULT_IOSCHED="deadline"` (lower latency than CFQ for flash storage)

### 6. `kernel/sched/sched_plus.h` — LB_CPU_MASK compile fix

Added `LB_POLICY_SHIFT` and `LB_CPU_MASK` to the `#else` block so `CONFIG_MTK_SCHED_BOOST`
code compiles even when `CONFIG_MTK_SCHED_TRACERS` is disabled.

### 7. `drivers/staging/android/lowmemorykiller.c` — missing include

Added `#include <linux/poll.h>` for `poll_table` type and `POLLIN` constant.

### 8. GPU `trace_events.c` — tracepoint redefinition fix

Wrapped function definitions in `drivers/misc/mediatek/gpu/gpu_rgx/m1.11ED5425693/services/server/env/linux/trace_events.c`
with `#ifdef CONFIG_EVENT_TRACING` to prevent redefinition conflicts with `static inline` stubs
in the corresponding header when `CONFIG_EVENT_TRACING` is not set.

### 9. VM sysctl tuning — hardcoded kernel defaults

Changed kernel defaults for better memory management on 2GB RAM device:

| Sysctl | Stock Default | New Default | File | Effect |
|--------|--------------|-------------|------|--------|
| `vm.vfs_cache_pressure` | 100 | 80 | `fs/dcache.c:84` | Keep dentries/inodes in cache longer |
| `vm.dirty_ratio` | 80 | 20 | `mm/page-writeback.c:91` | Start writeback sooner, less bursty I/O |

Other VM values already optimized by vendor: `swappiness=60`, `dirty_background_ratio=10`, `page-cluster=0`.

### 10. Other minor changes

- Deleted leftover `.rej` and `.bak` files from failed patch applications
- `sspm` Makefile: fixed `CONFIG_MTK_PLATFORM` quoting with `$(subst ",,...)`
- `cmdq/v3/Makefile`: fixed `MTK_PLATFORM` definition to strip quotes

## Build

```bash
# 1. Pull stock config from device
adb shell "cat /proc/config.gz | base64" > /tmp/config_b64.txt
python3 -c "import base64,gzip; open('.config','wb').write(gzip.decompress(base64.b64decode(open('/tmp/config_b64.txt').read())))"

# 2. Build
./build.sh

# 3. Flash (boot only — do NOT flash vbmeta separately)
adb reboot bootloader
fastboot flash boot new_boot.img
fastboot reboot
```

## What was NOT changed

- No driver logic changed (only build type: built-in → module)
- No DTS/DTSI files modified
- No SELinux policies changed
- No cmdline modifications in boot image
- Stock ramdisk used as-is (magiskboot repack)
- No debug configs disabled (MTK code has deep tracepoint dependencies that break if FTRACE/DEBUG_FS/TRACEPOINTS are turned off)

## Verification

After flashing, verify:
```bash
# Modules
adb shell "su -c lsmod"
# Should show: wmt_drv, wmt_chrdev_wifi, wlan_drv_gen4m, bt_drv, gps_drv, fmradio_drv

adb shell "su -c getprop vendor.connsys.driver.ready"
# Should show: yes

# I/O scheduler
adb shell "cat /sys/block/mmcblk0/queue/scheduler"
# Should show: [deadline]

# VM tuning (hardcoded in kernel, no su needed)
adb shell "cat /proc/sys/vm/vfs_cache_pressure"
# Should show: 80

adb shell "cat /proc/sys/vm/dirty_ratio"
# Should show: 20
```
