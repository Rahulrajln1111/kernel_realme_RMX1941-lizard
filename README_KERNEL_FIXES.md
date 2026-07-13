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

### 6. Other minor changes

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

## Verification

After flashing, verify all modules load:
```bash
adb shell "su -c lsmod"
# Should show: wmt_drv, wmt_chrdev_wifi, wlan_drv_gen4m, bt_drv, gps_drv, fmradio_drv

adb shell "su -c getprop vendor.connsys.driver.ready"
# Should show: yes
```
