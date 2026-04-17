# broadcom-sta-6.30.223.271-kernel617-patch

Created with help from Claude AI - Sonnet 4.6


A community patch to make the `broadcom-sta` (wl) wireless driver version 6.30.223.271 build and work correctly on Linux kernel 6.17+.

**Tested on:** Linux Mint 22.3 / Ubuntu 24.04 Noble, kernel 6.17.0-20-generic, MacBook Pro 11,1 (BCM4360)

---

## The Problem

When upgrading to kernel 6.17, the `broadcom-sta` DKMS module fails to build with errors including:

```
fatal error: typedefs.h: No such file or directory
error: implicit declaration of function 'from_timer'
error: implicit declaration of function 'del_timer'
memcpy: detected field-spanning write of single field "(&maclist->ea[i])"
error: unknown type name 'wl_iw_t'
error: initialization of 'int (*)(struct wiphy *, int, u32)' from incompatible pointer type
```

This is caused by multiple breaking changes in kernel 6.17:

- Kernel build system no longer resolves `broadcom-sta` include paths correctly
- `from_timer()` replaced by `timer_container_of()`
- `del_timer()` replaced by `timer_delete()`
- Binary blob `wlc_hybrid.o` no longer linked via `EXTRA_LDFLAGS`
- Fortified `memcpy` rejects writes to `ea[1]` fixed-size array in `struct maclist`
- `set_wiphy_params()` gained `int radio_idx` parameter
- `set_tx_power()` gained `int radio_idx` parameter
- `get_tx_power()` gained `int radio_idx` and `unsigned int link_id` parameters
- cfg80211 hybrid registration requires explicit `USE_CFG80211` flag

---

## What This Patch Does

- Fixes all include paths using absolute paths for the broadcom headers
- Adds kernel 6.17 version gates for changed timer API calls
- Fixes `struct maclist` to use flexible array member `ea[]`
- Fixes binary blob `wlc_hybrid.o` linking via `dkms.conf` MAKE command
- Fixes `set_wiphy_params`, `set_tx_power`, and `get_tx_power` function signatures
- Fixes implicit fallthrough warning in `wl_set_auth_type`
- Forces `USE_CFG80211` and `USE_IW` flags via `ccflags-y` in Makefile
- Adds `wl_iw.h` include to `wl_linux.h` to resolve `wl_iw_t` type

---

## How to Apply

### Prerequisites

```bash
sudo apt-get install broadcom-sta-dkms linux-headers-$(uname -r)
```

### Apply the patch

```bash
# Back up the original source
sudo cp -r /usr/src/broadcom-sta-6.30.223.271 /usr/src/broadcom-sta-6.30.223.271.bak

# Download the patch
wget https://raw.githubusercontent.com/snifferdogit/broadcom-sta-6.30.223.271-kernel617-patch/main/broadcom-sta-kernel617.patch

# Apply the patch
sudo patch -p0 -d /usr/src < broadcom-sta-kernel617.patch
```

### Build and install

```bash
sudo dkms build broadcom-sta/6.30.223.271 -k $(uname -r) --force
sudo dkms install broadcom-sta/6.30.223.271 -k $(uname -r) --force
sudo modprobe -r wl && sudo modprobe wl
```

### Verify

```bash
sudo dkms status
sudo iw dev
```

You should see `broadcom-sta/6.30.223.271` installed and your wireless interface registered.

---

## Affected Hardware

Any Linux system using the `broadcom-sta` (wl) proprietary driver, including:

- BCM4360
- BCM43602
- BCM4352
- BCM4331
- BCM43228
- BCM4322
- BCM43224/5
- BCM4313

---

## Kernel Compatibility

| Kernel | Status |
|--------|--------|
| 6.14.x | ✅ Working (original) |
| 6.17.x | ✅ Working (this patch) |

---

## Notes

- The patch maintains backward compatibility with older kernels via `#if LINUX_VERSION_CODE` guards
- The `wlc_hybrid.o` binary blob is pre-compiled by Broadcom and cannot be modified — the patch works around kernel 6.17's stricter linker requirements
- A udev rule may be needed to prevent the interface being renamed to `enp3s0` instead of `wlp3s0`

### Optional udev fix for interface naming

If your WiFi interface appears as `enp3s0` instead of `wlp3s0`, create this udev rule (replace the MAC address with yours from `ip link show`):

```bash
sudo tee /etc/udev/rules.d/70-broadcom-wifi.rules << 'EOF'
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="YOUR:MAC:HERE", NAME="wlp3s0"
EOF
sudo udevadm control --reload-rules
```

---

## Credits

Patch developed on Linux Mint 22.3 / Ubuntu 24.04 Noble, kernel 6.17.0-20-generic.  
GitHub: [snifferdogit](https://github.com/snifferdogit)

---

## Disclaimer

This patch is provided as-is. Always back up your original source before applying. The `broadcom-sta` driver is proprietary software owned by Broadcom — this patch only modifies the open-source wrapper code and build system files.
