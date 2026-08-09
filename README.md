

```
# GhostLock-Nevada – Motorola Moto G Play 2026 Port

**GhostLock (CVE-2026-43499)** exploit port for the **Motorola Moto G Play 2026 (XT2615-1)** – TracFone/Verizon variant.

This repository contains my work porting the GhostLock kernel exploit to the Motorola Moto G Play 2026. It's built on two existing projects:

- **[Root-My-Galaxy-Payloads](https://github.com/BuSung-dev/Root-My-Galaxy-Payloads)** – by BuSung-dev – the exploit framework that provides KASLR leak and escalation routes.
- **[UnPlus](https://github.com/No-22-Github/UnPlus)** – by No-22-Github – the blank template that gave the initial structure and `target.h` layout.

**Status:** ~90% complete. The exploit leaks memory, detects KASLR, runs the entire chain, but fails at the final CFI (Control Flow Integrity) check. See below for details.

---

## ⚠️ Current Status

**The exploit is ~90% complete.** It successfully:
- Leaks the `mm_struct`
- Detects KASLR (`slide-kaslr-ok`)
- Bypasses the `tracefs` check

**It fails at the final privilege escalation step** due to the kernel's **Control Flow Integrity (CFI)**:

```
[-] cfi misc_fops mismatch
[+] pipe physrw pid=... done=0 root=0 uid=4294967295->4294967295
```

Both the `pselect` and `pipe` routes hit the same CFI wall. This is likely due to a kernel mitigation that blocks function-pointer hijacking.

**If you are on a pre-June 2026 security patch**, this exploit may still work – please test and report back!

---

## 📱 Device Information

| Property | Value |
|---|---|
| **Device** | Moto G Play 2026 (XT2615-1) |
| **Carrier** | TracFone (Verizon) |
| **Build** | `W1WNS36.18-111-3` |
| **Kernel** | `5.15.189-android13-8-00018-g9c82b71884fd` |
| **Security Patch** | June 2026 |
| **Chipset** | MediaTek (MTK) – 39-bit VA |
| **Exploit Framework** | Root-My-Galaxy-Payloads (ported from UnPlus) |

---

## ✅ Working

- `boot.img` extraction from stock firmware.
- Symbol extraction using `vmlinux-to-elf` (159,345 symbols found).
- All critical kernel offsets (listed below).
- KASLR detection (`base=0xffffffc008000000`).
- `mm_struct` leak (`mm leaked=...`).
- `tracefs` check bypassed.
- Exploit compiles without errors (both `.so` and root binary).
- Both `pselect` and `pipe` routes run.

---

## ❌ Not Working

- **CFI bypass** – the kernel's Control Flow Integrity blocks the final function-pointer hijack.
- **Privilege escalation** – root is not achieved.

---

## 📂 Repository Structure

```
Root-My-Galaxy-Payloads/
├── src/
│   ├── targets/
│   │   ├── moto-g-play-2026/          # YOUR MOTOROLA PORT
│   │   │   ├── target.h               # All offsets filled in
│   │   │   └── p0_fingerprint.h       # Oracle constants
│   │   ├── pa3q-S938NKSUACZF1/        # S25 Ultra (CFI bypass reference)
│   │   ├── dm3q-S9180ZHS8FZF5/        # S23 Ultra
│   │   └── ... (other profiles)
│   ├── main.c
│   ├── util.c
│   ├── slide.c
│   ├── fops.c
│   ├── pipe.c
│   ├── root.c
│   └── preload.c
├── build/
│   └── moto-g-play-2026/
│       ├── cve-2026-43499             # LD_PRELOAD .so
│       └── cve-2026-43499-root        # Standalone binary
├── artifacts/                         # Pre-built binaries for other devices
├── support/                           # Target definitions
├── tools/                             # Helper scripts
├── Makefile
└── README.md
```

---

## 🔧 Offsets (for reference)

```c
#define KIMAGE_TEXT_BASE             0xffffffc008000000ULL
#define P0_PAGE_OFFSET               0xffffffc000000000ULL
#define P0_PHYS_OFFSET               0x40000000ULL
#define INIT_TASK_OFF                0x0000000002c43400ULL
#define INIT_CRED_OFF                0x0000000002bfd588ULL
#define ENTRY_TASK_OFF               0x0000000002ac82e8ULL
#define PER_CPU_OFFSET_OFF           0x0000000002b0cd58ULL
#define ROOT_TASK_GROUP_OFF          0x0000000002d58ac0ULL
#define SELINUX_ENFORCING_OFF        0x0000000002b4404ULL
#define SLIDE_NFULNL_LOGGER_NAME_OFF 0x0000000002b01e28ULL
#define SLIDE_SYSCTL_BOOTID_OFF      0x0000000002dc6819ULL
#define SYSTEM_UNBOUND_WQ_OFF        0x0000000002b007d8ULL
#define CALL_USERMODEHELPER_EXEC_WORK_OFF 0x000000000016f208ULL
#define ASHMEM_FOPS_OFF              0x0000000002106448ULL
#define ASHMEM_MISC_FOPS_OFF         0x0000000002c91b80ULL
```

Full `target.h` is in `src/targets/moto-g-play-2026/target.h`.

---

## 🔧 Build Instructions

### Prerequisites

- Android NDK r29 (or later)
- `adb` and `fastboot`
- Linux or WSL

### Build the Exploit

```bash
# Clone the repo
git clone https://github.com/crabcakes97/GhostLock-Nevada.git
cd GhostLock-Nevada

# Set NDK path
export ANDROID_NDK_HOME=~/android-ndk-r29

# Build for the Motorola target
make TARGET=moto-g-play-2026
```

### Outputs

The binaries will be in `build/moto-g-play-2026/`:
- `cve-2026-43499` – the LD_PRELOAD library
- `cve-2026-43499-root` – the standalone binary

### Push and Test

```bash
adb push build/moto-g-play-2026/cve-2026-43499 /data/local/tmp/
adb shell "LD_PRELOAD=/data/local/tmp/cve-2026-43499 RMG_FPSIMD_EXTERNAL_SIGNAL=1 RMG_DEBUG=1 id"
```

### Pre-built Binaries

If you don't want to compile, grab the pre-built binaries from the `build/moto-g-play-2026/` folder in the repo.

---

## 🧠 What I've Tried (Debugging History)

| Attempt | Result |
|---|---|
| Original UnPlus template | Failed at `pselect` (broken on 5.15) |
| Ported to Root-My-Galaxy-Payloads | Passed `tracefs`, failed at `KernelSnitch` |
| Enabled `RMG_FPSIMD_EXTERNAL_SIGNAL=1` | Leaked `mm_struct`, hit CFI |
| Updated all `ASHMEM_*` offsets | CFI persisted |
| Added `P0_ORACLE_*` constants | CFI persisted |
| Tried `pipe` route | Same CFI mismatch |
| Tried `pselect` route | Same CFI mismatch |
| Changed `P0_PHYS_OFFSET` | Same CFI mismatch |
| Rebooted / multiple attempts | No change |

**Conclusion:** The exploit gets to the final step but CFI blocks the write. This is a kernel-level mitigation that can't be bypassed by tweaking offsets – it requires a fundamentally different exploitation chain (e.g., ROP-based approach).

---

## 📋 Supported Devices

| Device | Kernel | Status |
|---|---|---|
| Moto G Play 2026 (XT2615-1) | 5.15.189 | ~90% (CFI blocker) |
| Other MTK variants | Unknown | Needs testing |

**This port is for MediaTek (MTK) devices only.** It will not work on Qualcomm/Snapdragon variants.

---

## 🔗 Related Profiles (for reference)

The `src/targets/` folder contains other profiles used as references:

| Profile | Device | Kernel | Purpose |
|---|---|---|---|
| `pa3q-S938NKSUACZF1` | Galaxy S25 Ultra | 6.6.98 | CFI bypass reference |
| `dm3q-S9180ZHS8FZF5` | Galaxy S23 Ultra | 5.15.189 | Same kernel version reference |
| `e1s-S921BXXSFDZF2` | Galaxy S24 | 6.1.xx | General reference |

These are not Motorola-compatible but were used to understand the exploit structure.

---

## 🙏 Credits

- **[Root-My-Galaxy-Payloads](https://github.com/BuSung-dev/Root-My-Galaxy-Payloads)** – by BuSung-dev – exploit framework and KASLR leak methods.
- **[UnPlus](https://github.com/No-22-Github/UnPlus)** – by No-22-Github – blank template that started this port.
- All developers in the GhostLock community who shared their debugging insights.

---

## 💬 How to Contribute

If you have a different carrier variant or can test on a pre-June patch, please share your results. If you have experience with CFI bypasses on 5.15 kernels, your help is highly appreciated.

Open an issue or submit a pull request.

---

## 📄 License

This project is for educational and research purposes only. Use at your own risk.

---

**GitHub:** https://github.com/crabcakes97/GhostLock-Nevada  
**XDA Thread:** https://xdaforums.com/t/dev-root-moto-g-play-2026-nevada-test.4797484/
```

---

## 🚀 Push to GitHub

```bash
cd ~/Root-My-Galaxy-Payloads
git add README.md
git commit -m "Update README with Motorola port, 90% status, new target folder, and all offsets"
git push
```

---

Your README now includes:
- The new `moto-g-play-2026` folder in the structure
- All the Motorola offsets
- 90% completion status with CFI blocker
- Build instructions specific to your target
- Credits to Root-My-Galaxy and UnPlus
- The debugging history
- References to other profiles in the repo
