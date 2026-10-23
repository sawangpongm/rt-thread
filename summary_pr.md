# RT-Thread Open PRs Summary

> Source: `https://github.com/RT-Thread/rt-thread/pulls?state=open`  
> Generated: 2026-10-23  
> Total open PRs: 21

---

## PR #11918 — `[dfs][9pfs] fix: clamp server-controlled lengths to the received reply`

| Field | Detail |
|-------|--------|
| **Author** | @usualtreee |
| **Branch** | `fix/9pfs-bounds` |
| **Created** | 2026-10-22 |
| **Category** | File System / 9PFS |

### Objective
Fix issue #11289: 9PFS client blindly trusts server-controlled length fields in `dfs_9pfs_read()` / `dfs_9pfs_getdents()`, causing out-of-bounds reads (ASAN use-after-poison, reading 8.3 MB past the receive buffer) and directory info leakage. The `p9_transaction()` return byte count was discarded, allowing a malicious/stale server to send a huge `count` value with a tiny actual packet.

### Solution
Clamp server-reported lengths (`count`, `stat_size`, `name_len`) against the actual received data size before parsing.

### Expected Output
- No out-of-bounds reads even with malicious 9P servers
- Directory enumeration stays within allocated buffer bounds
- ASAN-clean 9PFS operation

---

## PR #11916 — `测试一下`

| Field | Detail |
|-------|--------|
| **Author** | @H2S710 |
| **Branch** | `master` |
| **Created** | 2026-10-21 |
| **Category** | Test / (likely invalid) |

### Objective
Appears to be a test PR with no meaningful description (title translates to "Test it"). Likely a first-time contributor learning the GitHub PR workflow.

### Solution
N/A — no substantive changes described.

### Expected Output
N/A

---

## PR #11915 — `[dlmodule] fix: uninitialized globals in relocatable modules`

| Field | Detail |
|-------|--------|
| **Author** | @usualtreee |
| **Branch** | `fix/dlmodule-bss-common-overlap` |
| **Created** | 2026-10-21 |
| **Category** | Dynamic Module / ELF Loader |

### Objective
Fix issue #5705 (open since 2022): dynamic module loading on Cortex-M (ET_REL path) fails to properly handle uninitialized global variables. Four defects exist:

1. **BSS overlap**: After loading a SHT_NOBITS section, the allocation pointer doesn't advance, causing multiple BSS sections to overlap in the same memory
2. **Missing sh_addr correction**: Section symbol relocation formula lacks the section's link address
3. **SHN_COMMON skipped**: Tentative-definition uninitialized globals are silently skipped (wrong section index handling)
4. **No COMMON/BSS separation**: COMMON symbols get mixed with BSS without proper zeroing

### Solution
- Advance allocation pointer after each SHT_NOBITS (BSS) section
- Add `sh_addr` offset to section symbol relocations
- Properly detect and handle `SHN_COMMON` symbols (0xfff2)
- Zero-initialize both BSS and COMMON regions

### Expected Output
- Uninitialized globals in loaded modules have correct unique addresses
- No memory overlap / corruption across module boundaries
- Verified on Cortex-M BSPs with `RT_USING_MODULE`

---

## PR #11914 — `fix(libcpu/arm): correct format specifiers in GIC dump functions`

| Field | Detail |
|-------|--------|
| **Author** | @Wei-beep |
| **Branch** | `fix/arm-gic-format-specifiers` |
| **Created** | 2026-10-21 |
| **Category** | CPU / GICv2 & GICv3 |

### Objective
Fix `-Wformat` warnings in `arm_gic_dump_type()` and `arm_gic_dump()` across GICv2 (`gic.c`) and GICv3 (`gicv3.c`). Compiling Cortex-A BSPs with `-Wall -Wformat` produces:

- `%p` used with `rt_uint32_t` physical address (UB for non-pointers)
- `%d` used with unsigned values (signed/unsigned mismatch)
- Missing cast for GIC version extracted with `0xfUL`

### Solution
- Replace `%p` with `0x%08x` for 32-bit physical addresses
- Replace `%d` with `%u` for unsigned quantities
- Add explicit `(unsigned int)` cast for version fields

### Expected Output
- Clean build with `-Wall -Wformat` on all ARM Cortex-A BSPs
- No format specifier undefined behavior

---

## PR #11912 — `[components][ospi] Add OctaFlash Support`

| Field | Detail |
|-------|--------|
| **Author** | @CliffLL |
| **Branch** | `octaflash-driver` |
| **Created** | 2026-10-21 |
| **Category** | Components / OSPI / OctaFlash |

### Objective
Add OSPI (Octal SPI) support to RT-Thread's SPI driver layer and SFUD, enabling OctaFlash access beyond standard SPI and QSPI modes. Currently RT-Thread has no native OSPI path.

### Solution
- OSPI component layer API between SPI framework and SFUD
- Generic STM32 OctoSPI HAL driver (`drv_ospi.c`)
- Board-level OSPI Flash support for `stm32l4r9-st-eval` BSP
- SFUD integration for OctaFlash command set

### Expected Output
- OctaFlash chips readable/writable via SFUD on OSPI-capable boards
- Verified on STM32L4R9-ST-EVAL with onboard Macronix OctaFlash
- Existing SPI/QSPI paths remain unaffected

---

## PR #11911 — `[dfs][cromfs] fix: bound the directory name read by getdents`

| Field | Detail |
|-------|--------|
| **Author** | @Yunlk |
| **Branch** | `fix/cromfs-getdents-bound` |
| **Created** | 2026-10-21 |
| **Category** | File System / CROMFS |

### Objective
Fix issue #11905: `dfs_cromfs_getdents()` copies directory entry names using the raw `uint16 name_size` from the filesystem image without bounds checking:
- Exceeds `DIRENT_NAME_MAX` (256 bytes)
- No verification that the name fits within the loaded directory blob
- Writes NUL terminator past `d_name` buffer

### Solution
- Clamp `name_size` to `DIRENT_NAME_MAX - 1` before copy
- Verify name range is within the directory blob bounds
- Ensure NUL terminator always fits within `d_name[]`

### Expected Output
- No buffer overflow on malformed/corrupted CROMFS images
- Directory enumeration safe even with oversized names
- ASAN-clean CROMFS getdents

---

## PR #11910 — `[dfs][tmpfs] fix use-after-free when truncating a file to zero length`

| Field | Detail |
|-------|--------|
| **Author** | @remukisama |
| **Branch** | `fix/tmpfs-truncate-zero` |
| **Created** | 2026-10-21 |
| **Category** | File System / TMPFS |

### Objective
Fix issue #11902: calling `ftruncate(fd, 0)` on a non-empty tmpfs file causes `rt_realloc()` to free the old buffer and return `RT_NULL`. The original code returned success without clearing the data pointer or updating file length, leading to:
- Subsequent read/write accesses freed memory (use-after-free)
- File deletion may trigger memory assertion failures
- Non-zero truncate-to-smaller-size also wrongly returns success on allocation failure

### Solution
- Handle `offset == 0` separately: free buffer, clear pointer, then update length
- Non-zero offset still uses realloc but returns `-ENOMEM` on failure
- Add tmpfs utest cases for: repeated truncation, post-truncate R/W, deletion, non-zero shrink, and OOM handling

### Expected Output
- Zero-length truncation no longer leaks or use-after-free
- Failed non-zero truncation returns proper error code
- Verified by tmpfs unit tests

---

## PR #11909 — `[serial] fix: handle RT_SERIAL_EVENT_RX_TIMEOUT in the v1 receive path`

| Field | Detail |
|-------|--------|
| **Author** | @Yunlk |
| **Branch** | `fix/serial-rx-timeout` |
| **Created** | 2026-10-20 |
| **Category** | Drivers / Serial |

### Objective
Fix issue #8143: `RT_SERIAL_EVENT_RX_TIMEOUT` (0x05) is defined in `dev_serial.h` but `rt_hw_serial_isr()` only handles events 0x01~0x04 with no `default` branch. Two existing drivers (WCH CH579M, Synwit SYL9xx) depend on this event for idle-timeout frame delimiting — it is silently dropped.

### Solution
Add a case for `RT_SERIAL_EVENT_RX_TIMEOUT` in `rt_hw_serial_isr()` that performs the same notify-and-DMA-flush sequence as RX_DONE, allowing frame-based serial protocols to work with idle-timeout interrupts.

### Expected Output
- Serial drivers using idle-timeout frame detection emit proper events
- Frame-delimited protocols (RS-485, Modbus, etc.) work reliably
- CH579M and SYL9xx BSPs functional

---

## PR #11907 — `[kernel][scheduler] fix: keep FIFO order for same-priority threads`

| Field | Detail |
|-------|--------|
| **Author** | @Yunlk |
| **Branch** | `fix/8459-ready-list-order` |
| **Created** | 2026-10-20 |
| **Category** | Kernel / Scheduler |

### Objective
Fix issue #8459: `rt_sched_insert_thread()` inserts non-yielding threads at the head of the ready list. Threads woken from IPC pending queues don't have `RT_THREAD_STAT_YIELD` (cleared on every context switch), so FIFO order is reversed:

- A and B block on the same semaphore → pending queue: A → B
- Two releases wake A and B → ready list becomes B → A → B runs first

Newly started threads also get head-inserted, breaking "first started, first run" expectation.

### Solution
Append non-head-insert threads to the tail of the ready list instead of the head, preserving FIFO wakeup and startup order.

### Expected Output
- Same-priority threads execute in FIFO order after IPC wakeup
- Startup order matches creation order for equal-priority threads
- Verified on QEMU vexpress-a9 with semaphore tests

---

## PR #11903 — `[bsp][allwinner][d1s] add qemu-d1s.sh to run this BSP on the d1s QEMU machine`

| Field | Detail |
|-------|--------|
| **Author** | @cms19859230182-lang |
| **Branch** | `bsp-d1s-qemu-script` |
| **Created** | 2026-10-20 |
| **Category** | BSP / Allwinner D1s / QEMU |

### Objective
The D1s BSP currently only runs on real hardware. A companion project `d1s-qemu` (custom QEMU 10.1.2, ~500-line patch) adds a minimal D1s machine: C906 core, 64MB DRAM, UART0/UART3, PLIC, CLINT, SMHC0/SMHC1. This PR enables QEMU-based development without a physical board.

### Solution
- Add `bsp/allwinner/d1s/qemu-d1s.sh`: invokes `d1s-qemu/run.sh` with `sbi.bin`, `u-boot.dtb`, and compiled `rtthread.bin`, routing UART0 as console
- Update `bsp/allwinner/d1s/README.md` with QEMU setup instructions

### Expected Output
- D1s firmware buildable and runnable in QEMU for CI/testing
- No board required for BSP development

---

## PR #11901 — `[components][ofw]fix: add depth limit check for fdt_unflatten to prevent stack flow`

| Field | Detail |
|-------|--------|
| **Author** | @liangzhan5 |
| **Branch** | `add-depth-check` |
| **Created** | 2026-10-19 |
| **Category** | Components / Device Tree (OFW) |

### Objective
Fix issue #11879: `fdt_unflatten_single()` in `components/drivers/ofw/fdt.c` uses a fixed-size stack array `np_stack[OFW_NODE_MAX_DEPTH]` (default 64) to store traversal paths. Deeply nested DTB files (>64 levels) cause stack overflow before any depth check, leading to Data Abort or boot crash.

### Solution
- Add pre-check function `fdt_check_depth_limit()` before unflattening
- Manually traverse the DTB with a separate stack to compute max node depth
- If depth exceeds `OFW_NODE_MAX_DEPTH - 1` (63), halt and print error; skip unflattening

### Expected Output
- Malformed/deep DTBs rejected safely instead of crashing
- Boot-time stack protection for OFW subsystem
- Verified on QEMU

---

## PR #11900 — `[components][netdev] Fix ping maximum & minimum latency counting`

| Field | Detail |
|-------|--------|
| **Author** | @Huoyanlifusu |
| **Branch** | `fix/ping-latency-count` |
| **Created** | 2026-10-19 |
| **Category** | Components / Network / Ping |

### Objective
The `ping` utility incorrectly tracks max/min RTT latency across multiple pings due to flawed comparison logic (initialization or update order bug).

### Solution
Correct the initialization and comparison logic for min/max latency accumulation in the ping command handler.

### Expected Output
- Accurate min/max/avg latency statistics in `ping` output
- No spurious zero or overflowed latency values

---

## PR #11899 — `[components][lwp] Validate signal number in sys_sigaction`

| Field | Detail |
|-------|--------|
| **Author** | @BernardXiong |
| **Branch** | `fix/lwp-sigaction-signo` |
| **Created** | 2026-10-19 |
| **Category** | Components / LWP / Signal |

### Objective
Fix issue #11882: `sys_sigaction()` passes user-space `sig` directly to `lwp_signal_action()` without range checking. Since `lwp_signal_action()` indexes `sig_action[]` / `sig_action_mask[]` arrays (size `_LWP_NSIG = 64`) with `signo - 1`, out-of-range or negative signals cause kernel heap out-of-bounds read/write. An unprivileged user-space program can trigger:

- Arbitrary 8-byte write via user-controlled handler offset
- Kernel heap info leak via `oact` copy-back
- Kernel crash on far-out-of-bounds access

### Solution
- Validate `signo` in `sys_sigaction()` entry: accept only `[1, _LWP_NSIG]`
- Return `EINVAL` for invalid signals before any array access

### Expected Output
- Signal numbers bounds-checked at syscall boundary
- No kernel heap corruption from malformed `sigaction()` calls
- LWP security hardening

---

## PR #11897 — `[bsp][k230] add K230D support and Smart improvements`

| Field | Detail |
|-------|--------|
| **Author** | @BernardXiong |
| **Branch** | `pr/k230-k230d-smart` |
| **Created** | 2026-10-18 |
| **Category** | BSP / Kendryte K230 / Smart |

### Objective
The K230 BSP only targeted the 512MB K230 board. The K230D (128MB DDR) variant was unsupported. Also missing: GPIO pin device, ext4 rootfs, and pseudo-terminal mount points required by RT-Thread Smart.

### Solution
- Add `SOC_K230` / `SOC_K230D` Kconfig selection with 128MB DDR memory layout
- Register GPIO as an RT-Thread pin device
- Add ext4 rootfs support, mount `ptyfs` and `procfs`
- Smart configuration integration

### Expected Output
- K230D board bootable with Smart user-space features
- GPIO pin API available
- Standard rootfs layout for Smart applications

---

## PR #11896 — `[bsp][qemu-virt64] enable RT-Thread Smart`

| Field | Detail |
|-------|--------|
| **Author** | @BernardXiong |
| **Branch** | `pr/qemu-virt64-smart` |
| **Created** | 2026-10-18 |
| **Category** | BSP / QEMU Virt64 / Smart |

### Objective
The `qemu-virt64` BSPs (RISC-V and AArch64) did not enable RT-Thread Smart, preventing user-space and LWP testing under QEMU. No standard rootfs initialization existed for the virtio block device or pseudo terminals.

### Solution
- Enable `RT_USING_SMART` for both `qemu-virt64-riscv` and `qemu-virt64-aarch64`
- Disable SMP on riscv64 to match Smart single-core configuration
- Add init routine for virtio-blk rootfs and pseudo terminal setup

### Expected Output
- RT-Thread Smart runnable in QEMU on both RISC-V and AArch64 targets
- User-space processes (LWP) testable without physical hardware
- Standard block device + pty initialization

---

## PR #11895 — `[components][lwp] fix signal ordering and ELF segment protection`

| Field | Detail |
|-------|--------|
| **Author** | @BernardXiong |
| **Branch** | `pr/lwp-signal-elf` |
| **Created** | 2026-10-18 |
| **Category** | Components / LWP / Signal & ELF |

### Objective
Three LWP bugs:

1. Same-number signals inserted in reverse order (newest-first instead of FIFO)
2. ELF segments always mapped R/W regardless of `p_flags` (security issue)
3. RISC-V signal handling disabled interrupts while acquiring sleeping locks (deadlock risk)

### Solution
- Maintain FIFO insertion order for pending signals with the same number
- Derive segment permissions from ELF `p_flags` (R/W/X as declared)
- Keep scheduler functional during RISC-V signal handling (don't mask interrupts around sleeping locks)

### Expected Output
- Signal delivery order predictable and correct
- ELF memory protection enforced (W^X where applicable)
- No deadlock during signal delivery on RISC-V

---

## PR #11894 — `[components][lwp] split syscall implementation by subsystem`

| Field | Detail |
|-------|--------|
| **Author** | @BernardXiong |
| **Branch** | `pr/lwp-syscall-split` |
| **Created** | 2026-10-18 |
| **Category** | Components / LWP / Syscall Refactor |

### Objective
`lwp_syscall.c` is a single 11,000+ line file covering memory, filesystem, socket, process, IPC, and time syscalls. It's impossible to navigate, review, or maintain.

### Solution
- Move syscall implementations into `components/lwp/syscall/` directory
- Group by subsystem: `memory/`, `file/`, `socket/`, `process/`, `ipc/`, `time/`
- Add shared syscall dispatch table and internal header
- Register new files in the build system (SConscript / CMake)

### Expected Output
- Each subsystem in its own file (~1-2K lines each)
- Easier code review and parallel development
- No functional change — pure structural refactor

---

## PR #11893 — `[net][lwip] add loopback netdev support`

| Field | Detail |
|-------|--------|
| **Author** | @BernardXiong |
| **Branch** | `pr/lwip-loopback` |
| **Created** | 2026-10-18 |
| **Category** | Network / lwIP / Loopback |

### Objective
lwIP's loopback interface (`127.0.0.1`) was not registered as an RT-Thread `netdev`, causing:
- Tools relying on localhost to fail
- No default interface in network namespaces without a physical NIC
- `ping 127.0.0.1` and socket loopback tests unavailable

### Solution
- Add `RT_LWIP_USING_LOOPBACK` Kconfig option
- Register `lo` / `lo0` as a netdev with link-up state
- Select loopback as default interface when no physical link exists
- Increase RAW receive mailbox size for loopback echo

### Expected Output
- `127.0.0.1` pingable and socket-connectable via netdev API
- Network stack functional even without physical Ethernet/Wi-Fi
- Namespace-local communication works

---

## PR #11892 — `[libcpu][riscv] fix MMU and user fault handling for Smart`

| Field | Detail |
|-------|--------|
| **Author** | @BernardXiong |
| **Branch** | `pr/riscv-smart-mmu` |
| **Created** | 2026-10-18 |
| **Category** | libcpu / RISC-V / MMU / Smart |

### Objective
Multiple RISC-V Smart issues broke reliable user-space operation:

1. Early page table pointers not translated with kernel physical offset (wrong memory map)
2. Per-hart page tables inherited user mappings (security leak between harts)
3. Early page tables not flushed after setup (stale TLB entries)
4. Unhandled user faults terminated the **kernel** instead of the **current process** (system-wide crash)

### Solution
- Add XuanTie architecture selection and SVPBMT (Page-Based Memory Type) option
- Translate early page table addresses with kernel physical offset
- Isolate per-hart page tables; flush early tables after initialization
- Route unhandled user faults to process termination (SIGSEGV) instead of kernel panic

### Expected Output
- User-space processes isolated across harts
- Page tables correctly mapped with physical offset
- Faulty user processes killed individually; kernel survives
- Smart-enabled RISC-V BSPs stable under user load

---

## PR #11891 — `[drivers][serial] improve TIOCGWINSZ and termios updates`

| Field | Detail |
|-------|--------|
| **Author** | @BernardXiong |
| **Branch** | `pr/serial-tiocgwinsz` |
| **Created** | 2026-10-18 |
| **Category** | Drivers / Serial / Termios |

### Objective
Two serial terminal bugs:

1. **TIOCGWINSZ** (terminal window size query) reads the CSI `18 t` response via `finsh_getchar()`, which only works in the `tshell` thread and can consume unrelated input
2. **Termios updates** reconfigure the UART even when no hardware settings changed, resetting the FIFO and causing data loss

### Solution
- Write CSI `18 t` query directly to the serial device, bypassing `finsh`
- Read response one byte at a time with a 200 ms timeout
- Strictly parse the `CSI 8 ; rows ; cols t` response format
- Skip UART reconfiguration if termios fields match current state

### Expected Output
- TIOCGWINSZ works in any thread context
- No spurious UART FIFO resets on trivial termios changes
- Terminal resize detection reliable in Smart user-space

---

# Summary by Category

| Category | PRs | Themes |
|----------|-----|--------|
| **File System** | #11918, #11911, #11910 | 9PFS bounds, CROMFS bounds, TMPFS use-after-free |
| **Kernel / Scheduler** | #11907 | FIFO ready-list order |
| **Kernel / LWP** | #11899, #11895, #11894 | Signal validation, ordering, ELF protection, syscall refactor |
| **Network** | #11893, #11900 | lwIP loopback netdev, ping latency counting |
| **Drivers / Serial** | #11909, #11891 | RX timeout event, TIOCGWINSZ, termios |
| **CPU / Architecture** | #11914, #11892 | ARM GIC format fixes, RISC-V MMU+fault handling |
| **BSP** | #11912, #11903, #11897, #11896 | OctaFlash, D1s QEMU, K230D, qemu-virt64 Smart |
| **Components / OFW** | #11901 | FDT depth limit check |
| **Dynamic Modules** | #11915 | ELF relocatable module BSS/COMMON fix |
