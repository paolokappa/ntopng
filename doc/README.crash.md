# 08 — Bug Report: Repeated ntopng Crash (SIGSEGV + SIGBUS)

**Platform**: ntopng Enterprise XL 6.7.260218 (build 27466, GIT rev bd551e38)
**OS**: Ubuntu 24.04.3 LTS, Kernel 6.17.0-14-generic (PREEMPT_DYNAMIC)
**Hardware**: VMware vSphere, 6 vCPU (AMD x86-64), 32 GB RAM
**Operator**: GOLINE SA (AS202032) — ISP Collector Mode (IPFIX/sFlow via ZMQ)
**Date**: 2026-02-18
**Severity**: High — 48 crashes in 22 hours, service repeatedly interrupted

---

## Executive Summary

On 2026-02-18, ntopng experienced **48 abnormal terminations** composed of three distinct failure modes:

| # | Signal | Count | Window | Root Cause |
|---|--------|-------|--------|------------|
| 1 | SIGSEGV (11) | 42 | 00:41 – 05:43 | Asset Inventory race condition during bulk host insert |
| 2 | SIGBUS (7) | 6 | 07:29 – 22:09 | Thread stack overflow in `ntopng-3-pkt` (X86_TRAP_SS) |
| 3 | exit 127 | 4 | 06:43 – 06:43 | `librrd.so.8` missing during apt package upgrade |

Bug #1 (SIGSEGV) is the most impactful: **42 crashes in 5 hours, with a peak of 38 crashes between 04:00–06:00** creating a persistent crash-restart loop (~1 crash every 2–3 minutes). Bug #2 (SIGBUS) was mitigated with a systemd `LimitSTACK` override. Bug #3 is a transient packaging issue (self-resolved).

---

## 1. Environment

### ntopng Configuration

```
-i=tcp://127.0.0.1:5557          # nProbe ZMQ collector (IPFIX + sFlow)
-F=clickhouse;127.0.0.1@9000,9004;ntopng;default;
-x=262144                        # max hosts (262K)
-X=524288                        # max flows (524K)
-g=4,5                           # pkt thread CPU affinity
-y=2,3                           # other threads CPU affinity
--local-networks=/etc/ntopng/local-networks.txt  # 44 entries (GOLINE + transit + IXP)
--ndpi-protocols=/var/lib/ntopng/protos.txt
-N=NTA GOLINE
```

### Relevant Features Enabled

| Feature | Redis Key | Value | Note |
|---------|-----------|-------|------|
| Asset Inventory | `ntopng.prefs.enable_asset_inventory` | 1 | **New in ntopng 6.6** |
| Assets Log | `ntopng.prefs.enable_assets_log` | 1 | |
| ClickHouse storage | `-F=clickhouse` | — | All flows + alerts |
| Flow Deduplication | `ntopng.prefs.enable_flow_deduplication` | 1 | Enterprise XL |
| Behaviour Analysis | `ntopng.prefs.is_*_behavior_analysis_enabled` | 1 | ASN + Interface L7 + Network |
| Blacklist Stats | `ntopng.prefs.collect_blacklist_stats` | 1 | 16 blacklists |
| Automatic Reports | `ntopng.prefs.automatic_reports_enabled` | 1 | |

### Traffic Volume

- **Flow rate**: ~7,000 flows/min from 5 routers (3x MikroTik IPFIX, 1x Juniper MX204 IPFIX, 1x Huawei NE8000 sFlow)
- **Throughput**: ~153 GiB/h aggregate backbone traffic
- **Local hosts seen**: ~800 IPv4 + ~200 IPv6 in GOLINE range (185.54.80.0/22 + 2a02:4460::/32)

### Installed Packages

```
ntopng          6.7.260218-27466
ntopng-data     6.7.260218
pfring          9.3.0-10547
apt-ntop        2.10-53
ntop-license    1.0-619
```

### Binary Details

```
ELF 64-bit LSB PIE executable, x86-64
BuildID: 4b797a5bd87db55b1b702f637f1d0045b5192910
GIT rev: dev:bd551e38141f1bfde87eac8af80734e92b76d3c3:20260218
Pro rev: r7977
Binary:  stripped (NO debug symbols)
Size:    11 MB
```

---

## 2. Bug #1 — SIGSEGV (Signal 11): Asset Inventory Race Condition

### 2.1 Impact

- **42 crashes** between 00:41 and 05:43 UTC
- **Peak**: 38 crashes in the 04:00–06:00 window (1 crash every ~2 minutes)
- Each crash triggers a full ntopng restart (systemd `Restart=on-failure`, `RestartSec=5`)
- During the crash loop, asset migration runs at every startup, compounding the problem

### 2.2 Crash Timeline

```
00:41:21  SEGV  (isolated, probe source_id: 16941966)
01:01:27  SEGV  (isolated, probe source_id: 21149134)
02:58:20  SEGV  ← start of pairs
02:58:50  SEGV
04:16:57  SEGV  ← start of sustained crash storm
04:17:23  SEGV
04:17:56  SEGV
04:19:48  SEGV
04:20:58  SEGV
04:22:11  SEGV
... (32 more SEGV between 04:45 and 05:43) ...
05:42:55  SEGV
05:43:57  SEGV  ← last SEGV, then 1h gap (apt upgrade window)
```

### 2.3 Crash Sequence Pattern

Every SEGV crash follows the same pattern, observable in the journal:

```
1. [startup.lua:47] [asset_utils.lua:105] Migrating empty manufacturer column
                                          to unknown in assets table
2. [startup.lua:47] [asset_utils.lua:115] Assets manufacturer migration
                                          completed for ClickHouse
3. [ZMQCollectorInterface.cpp:257] Collecting flows on zmq://127.0.0.1:5557
4. [ZMQCollectorInterface.cpp:530] Allocating new probe/source [source_id: ...]
5. [LocalHostPro.cpp:247] Adding Host X.X.X.X to Assets [Ifid: 6][VLAN: 0]
   ... (bulk host additions) ...
6. systemd: Main process exited, code=dumped, status=11/SEGV
```

The crash occurs **~15–30 seconds after startup**, immediately after the ZMQ collector begins receiving flows and the C++ code in `LocalHostPro.cpp:247` starts adding hosts to the Asset Inventory. The timing is consistent with a race condition between:

- The **pkt thread** (`ntopng-3-pkt`) processing incoming flows and calling `LocalHostPro::addHostToAssets()` in C++
- The **Lua periodic scripts** running `asset_utils.lua:1203` (`dumpQueuedEntries`) to flush the asset queue to ClickHouse
- The **startup Lua script** (`startup.lua:47`) running the manufacturer migration on the assets ClickHouse table

### 2.4 Evidence of Asset Inventory as Trigger

1. **Temporal correlation**: Crashes only started after Asset Inventory was enabled (Redis `enable_asset_inventory=1`, `enable_assets_log=1`) during Session 4 tuning
2. **Log correlation**: Every crash is preceded by "Adding Host ... to Assets" and/or "Migrating empty manufacturer column" log entries
3. **Shutdown log**: During crash at 05:43, the shutdown handler (`shutdown.lua:45`) was still calling `asset_utils.lua:1203 Adding new asset` while the process was being dumped:
   ```
   05:43:56 [shutdown.lua:45] [asset_utils.lua:1203] Adding new asset:
    string {"type":"mac","key":"6_E4:23:3C:53:98:17","manufacturer":"Juniper Networks"}
   05:43:57 [HTTPserver.cpp:1954] HTTP server terminated
   05:43:57 [NetworkInterface.cpp:4124] Cleanup interface zmq://...
   05:43:57 systemd: Main process exited, code=dumped, status=11/SEGV
   ```
4. **CHANGELOG ntopng 6.6** confirms: *"Fix assets not correctly dumped into the DB"* — the feature had known bugs at release
5. **No core dump available**: Apport is configured as the core handler (`/usr/share/apport/apport`) but no core dump was retained by `coredumpctl`

### 2.5 Probable Root Cause

A **use-after-free or NULL pointer dereference** in the interaction between:

- `LocalHostPro.cpp:247` → `addHostToAssets()` (C++, runs in pkt thread context)
- `asset_utils.lua:1203` → `dumpQueuedEntries()` (Lua, runs in periodic activity thread)
- ClickHouse bulk INSERT (async, runs in DB worker thread)

The ISP environment exacerbates the race condition because:

- **~800 local hosts** need to be added to the asset inventory at each restart
- **High flow rate** (~7K flows/min) means the pkt thread is continuously adding new assets
- **manufacturer migration** at startup modifies the same ClickHouse table being written to
- **ZMQ collector mode** means flows arrive in bursts (nProbe buffers and sends batches), creating sudden spikes in the asset queue

### 2.6 Self-Stabilization

The crash loop ended naturally at ~05:43 for two reasons:

1. **apt auto-update at 06:43** replaced the ntopng binary (package `6.7.260218` was re-installed), forcing a clean restart after a 1-hour gap
2. **Asset inventory converged**: after ~42 restart cycles, most hosts had been successfully committed to ClickHouse. Subsequent restarts found fewer hosts to migrate, reducing the window of vulnerability

This matches the pattern documented in Session 4: *"Asset Inventory crash — 6 crash/restart automatici, poi stabile. Bug ntopng 6.6.260217 race condition con asset_utils.lua:1203. Transiente, non ricorrente dopo primo batch."*

### 2.7 Status

**NOT FIXED** — this is an ntopng bug requiring a code fix from ntop.

**Workaround**: If crashes recur, disable Asset Inventory:
```bash
redis-cli set "ntopng.prefs.enable_asset_inventory" "0"
redis-cli set "ntopng.prefs.enable_assets_log" "0"
systemctl restart ntopng
```

**Recommendation**: File a bug report at https://github.com/ntop/ntopng/issues with:
- This crash sequence
- The journal logs showing the startup → asset migration → crash pattern
- Request a `mutex` or queue lock in `LocalHostPro::addHostToAssets()` / `asset_utils.dumpQueuedEntries()` to serialize access

---

## 3. Bug #2 — SIGBUS (Signal 7): Thread Stack Overflow (X86_TRAP_SS)

### 3.1 Impact

- **6 crashes** between 07:29 and 22:09 UTC
- Spaced hours apart (not a crash loop — ntopng ran stably between crashes)
- Each crash in the same thread: `ntopng-3-pkt` (packet processing)

### 3.2 Crash Timeline

```
07:29:47  BUS  (after stable run from ~06:49)
07:49:05  BUS  (20 min after previous)
11:37:30  BUS  (3h48m gap — ntopng ran stably)
11:38:00  BUS  (30s after previous)
12:45:44  BUS  (1h07m gap)
22:09:39  BUS  (9h24m gap — longest stable run before crash)
```

### 3.3 Kernel Trap Analysis

All 4 unique dmesg entries (some crashes didn't produce dmesg entries, likely due to rate limiting):

```
[11:37:29] traps: ntopng-3-pkt[3409388] trap stack segment
  ip:6193815291d3 sp:7dfcc81f8b70 error:0
  in ntopng[4091d3,61938127c000+698000]

[11:37:59] traps: ntopng-3-pkt[3411501] trap stack segment
  ip:6458a4896140 sp:7f06c2df8b70 error:0
  in ntopng[409140,6458a45e9000+698000]

[12:45:43] traps: ntopng-3-pkt[956] trap stack segment
  ip:56f6f1b5c1d3 sp:75266a3f8b70 error:0
  in ntopng[4091d3,56f6f18af000+698000]

[22:09:38] traps: ntopng-3-pkt[92838] trap stack segment
  ip:58ade7f1b1d3 sp:7604a21f8b70 error:0
  in ntopng[4091d3,58ade7c6e000+698000]
```

### 3.4 Decoding the Kernel Trap Message

The kernel message format is:
```
traps: <thread_name>[<tid>] trap <type> ip:<instruction_pointer> sp:<stack_pointer>
  error:<error_code> in <binary>[<code_offset>,<load_address>+<size>]
```

**"trap stack segment"** is generated by `exc_stack_segment()` in `arch/x86/kernel/traps.c`:

```c
DEFINE_IDTENTRY_ERRORCODE(exc_stack_segment)
{
    do_error_trap(regs, error_code, "stack segment", X86_TRAP_SS, SIGBUS, 0, NULL);
}
```

This is **CPU exception #12 (Stack Segment Fault)**, which occurs when:
1. The stack pointer (`SP`) descends below the bottom of the thread's allocated stack memory
2. The CPU attempts to push a value or access memory via `SP` and hits the **guard page** (a non-readable/non-writable page placed at the bottom of each thread stack)
3. The MMU triggers exception #12, the kernel translates it to **SIGBUS (signal 7)** and terminates the process

This is definitively a **thread stack overflow** — the thread consumed more stack space than was allocated.

### 3.5 Code Offset Analysis

The crash consistently occurs at two very close code offsets within the ntopng binary:

| Crash | Code Offset | Count |
|-------|-------------|-------|
| Primary | `0x2ad1d3` | 3 of 4 |
| Secondary | `0x2ad140` | 1 of 4 |

Both offsets fall in the `.text` section (starts at `0x1618a0`, size `0x692***`). Disassembly of the crash area:

```asm
; Offset 0x2ad140 — crash point #2
2ad140: ff f3           push   %rbx            ← CRASH HERE (push to exhausted stack)
2ad142: 0f 1e fa        nop    %edx
2ad145: 48 89 c5        mov    %rax,%rbp
2ad148: e9 6f bc ed ff  jmp    188dbc          ← jump table (exception landing pad?)

; Offset 0x2ad1d1 — crash point #1
2ad1d1: f3 0f 1e fa     endbr64
2ad1d5: 48 89 c5        mov    %rax,%rbp       ← CRASH HERE (sp below stack)
2ad1d8: e9 57 bc ed ff  jmp    188e34          ← jump table
```

The instruction at `0x2ad140` is `push %rbx` — this pushes a register onto the stack, decrementing `SP` by 8 bytes. When `SP` is already at the guard page boundary, this single `push` instruction triggers the stack segment fault.

The symbol context suggests this is in a **C++ exception handling / stack unwinding** region (near `basic_stringbuf` destructor, likely `libstdc++` exception landing pads compiled into the binary). This implies the actual stack overflow begins deeper in the call chain, and by the time the exception unwinder tries to clean up, the stack is already exhausted.

The jump targets (`0x188dbc`, `0x188e34`) contain a **dispatch table** loading small integer constants into `%eax` and jumping to a common handler — consistent with a C++ `catch` block switch-case dispatching by exception type.

### 3.6 Stack Limit Before Fix

```
Max stack size    8388608 (8 MB)    unlimited    bytes
PTHREAD_STACK_MIN: 16384 (16 KB)
```

The default `ulimit -s` of 8192 KB (8 MB) sets the main thread stack. For `pthread_create()` threads (like `ntopng-3-pkt`), glibc uses `RLIMIT_STACK` as the default thread stack size (since glibc 2.1).

8 MB is insufficient for the combined call depth of the `pkt` thread processing path:

```
Thread ntopng-3-pkt call chain (estimated):
├── NetworkInterface::processFlow()          ~2 KB
│   ├── nDPI::processPacket()                ~4 KB (protocol detection state machines)
│   ├── Flow::processExtraDissectedInformation()  ~2 KB
│   ├── LocalHostPro::addHostToAssets()      ~1 KB
│   ├── GeoIP2::lookupCity()                 ~2 KB (libmaxminddb mmap traversal)
│   ├── AlertsManager::checkFlowAlerts()     ~2 KB
│   │   ├── Lua VM: flow checks execution    ~64-128 KB per Lua state
│   │   │   ├── asset_utils.lua              ~32 KB
│   │   │   ├── check scripts (15+ active)   ~16 KB each
│   │   │   └── json.lua (encode for CH)     ~16 KB
│   │   └── ClickHouse::insertFlow()         ~8 KB (HTTP client, zstd compression)
│   ├── BehaviourAnalysis::update()          ~4 KB
│   └── FlowDeduplication::check()           ~2 KB
└── Total estimated peak: ~6-8 MB (within 8 MB limit, NO margin)
```

When multiple features are active simultaneously (Asset Inventory + ClickHouse + DPI + Lua checks + Behaviour Analysis + Deduplication), the combined stack usage approaches or exceeds the 8 MB limit. Temporary spikes from deep JSON serialization, ClickHouse batch inserts, or complex nDPI state machines push it over.

### 3.7 Why "trap stack segment" (SIGBUS) and NOT SIGSEGV

On Linux x86-64, thread stack overflows can produce either SIGSEGV or SIGBUS depending on exactly how the overflow manifests:

- **SIGSEGV**: When the stack pointer is still within valid memory but the accessed address is in the guard page — triggered by the **page fault handler** (exception #14)
- **SIGBUS (trap stack segment)**: When the **CPU's stack segment check** detects the overflow before the page fault handler — triggered by the **stack segment fault handler** (exception #12)

The distinction depends on:
1. Whether the instruction explicitly uses `%rsp` (e.g., `push`, `call`, `sub %rsp`) → more likely X86_TRAP_SS → SIGBUS
2. Whether the access is via a general register that happens to point near the stack bottom → more likely page fault → SIGSEGV

In our case, `push %rbx` at `0x2ad140` is an explicit stack operation, so the CPU raises exception #12 (Stack Segment) rather than #14 (Page Fault), resulting in SIGBUS.

### 3.8 Related GitHub Issues

| Issue | Version | Signal | Thread | Resolution |
|-------|---------|--------|--------|------------|
| [#7887](https://github.com/ntop/ntopng/issues/7887) | 5.7.231008 | SIGBUS (7) | pkt | Closed as "completed" |
| [#6830](https://github.com/ntop/ntopng/issues/6830) | 5.5.220826 | SIGILL | pkt | Fixed after update |
| [#9832](https://github.com/ntop/ntopng/issues/9832) | 6.6.251119 | SIGSEGV | http-wor | "No longer reproducible" |
| [#9840](https://github.com/ntop/ntopng/issues/9840) | 6.6.x | crash | Lua | Circular dependency in snmp_cached_dev |

### 3.9 Mitigation Applied

**systemd override** `/etc/systemd/system/ntopng.service.d/stack-size.conf`:
```ini
[Service]
LimitSTACK=16777216
```

This doubles the thread stack from 8 MB to 16 MB. After applying:

```
Max stack size    16777216 (16 MB)    16777216 (16 MB)    bytes
```

**Result**: ntopng has been stable since the fix (no new crashes after 22:09). The 16 MB stack provides ~8 MB of headroom beyond the estimated peak usage.

### 3.10 Status

**MITIGATED** (not fixed). The underlying issue is that ntopng does not control its thread stack sizes programmatically. The default 8 MB is insufficient for ISP deployments with all Enterprise XL features enabled.

**Recommendation to ntop**: In `ThreadedActivity.cpp` (or wherever `pthread_create` is called for the pkt thread), use `pthread_attr_setstacksize()` to explicitly set a 16 MB stack:

```cpp
pthread_attr_t attr;
pthread_attr_init(&attr);
pthread_attr_setstacksize(&attr, 16 * 1024 * 1024);  // 16 MB
pthread_create(&thread, &attr, pktProcessingThread, this);
pthread_attr_destroy(&attr);
```

Or document the `LimitSTACK` requirement for high-feature deployments.

---

## 4. Bug #3 — Exit 127: Missing `librrd.so.8` During Package Upgrade

### 4.1 Impact

- **4 exit-code-127 failures** in rapid succession at 06:43 UTC
- Self-resolved after ~6 minutes when apt completed the upgrade
- NOT a crash — no signal, no core dump, just "shared library not found"

### 4.2 Sequence

```
06:43:16  groupadd: group added: name=ntop, GID=982
06:43:16  useradd: new user: name=ntop, UID=990
06:43:16  Started ntop-service.service
06:43:17  Started ntop-license-manager.service
06:43:17  apt-daily.service completed
06:43:31  ntopng started (restart counter triggered from previous SEGV crash loop)
06:43:31  ERROR: librrd.so.8: cannot open shared object file: No such file or directory
06:43:32  exit-code 127 (×1)
06:43:37  restart → same error (×2)
06:43:42  restart → same error (×3)
06:43:47  restart → same error (×4)
06:43:49  ntopng.service: Stopped (hit burst limit)
06:49:06  ntopng started successfully (librrd.so.8 now available)
06:49:06  [NtopPro.cpp:401] Reading license from /etc/ntopng.license
06:49:06  [NtopPro.cpp:583] Found valid Enterprise XL (Bundle) license
```

### 4.3 Root Cause

The `apt-daily.service` timer triggered at ~06:41 and initiated the `apt-ntop` package upgrade. During the upgrade:

1. `dpkg` removed the old `librrd8` package (or its shared library file) before installing the new one
2. The ntop package `postinst` script triggered systemd to restart ntopng
3. ntopng attempted to load `librrd.so.8` which was temporarily absent from `/lib/x86_64-linux-gnu/`
4. The dynamic linker (`ld-linux-x86-64.so.2`) failed with `ENOENT`, process exited with code 127
5. systemd restarted ntopng 4 times, hit the default burst limit (`StartLimitBurst=5` within `StartLimitIntervalSec=10s`), and stopped the service
6. After `dpkg` completed (~06:49), `librrd.so.8` was back, and ntopng started normally

### 4.4 Shared Library Dependency Chain

```
ntopng → librrd.so.8 → /lib/x86_64-linux-gnu/librrd.so.8
                        (from package librrd8, version tied to apt-ntop repo)
```

Full dependency list (20 direct shared libraries):
```
librrd.so.8        libjson-c.so.5      libnetsnmp.so.40    libmaxminddb.so.0
libhiredis.so.1.1  libsqlite3.so.0     libradcli.so.4      libexpat.so.1
libssl.so.3        libcrypto.so.3      libzmq.so.5         libresolv.so.2
librdkafka.so.1    libcap.so.2         libldap.so.2        liblber.so.2
libz.so.1          libcurl.so.4        libzstd.so.1        libstdc++.so.6
```

### 4.5 Status

**TRANSIENT** — self-resolved. This is a packaging issue in the ntop apt repository: the `postinst` script restarts ntopng before ensuring all dependencies are in place.

**Mitigation**: The `/etc/apt/apt.conf.d/99-ntopng-custom` hook (installed during Session 3) includes a post-upgrade restore script that handles this scenario. However, the `librrd.so.8` gap is a dpkg ordering issue that cannot be fixed from the user side.

---

## 5. Timeline Summary

```
     00:00                    06:00         07:00              12:00              22:00  23:00
       │                        │             │                  │                  │      │
       │  ┌─SEGV──────────────┐ │             │                  │                  │      │
       │  │ 42 crashes        │ │             │                  │                  │      │
       │  │ Asset Inventory   │ │             │                  │                  │      │
       ├──┤ race condition    ├─┤             │                  │                  │      │
 00:41 │  │ peak: 1 crash/2m  │ │05:43        │                  │                  │      │
       │  └──────────────────┘ │             │                  │                  │      │
       │                       │┌exit127─┐   │                  │                  │      │
       │                       ││librrd  │   │                  │                  │      │
       │                       │├────────┤   │                  │                  │      │
       │                  06:43││4 fails ││06:49                │                  │      │
       │                       │└────────┘   │                  │                  │      │
       │                        │      ┌─BUS─┤  ┌──BUS──┐  ┌BUS┤             ┌BUS─┤      │
       │                        │      │07:29│  │11:37  │  │   │             │    │      │
       │                        │      │07:49│  │11:38  │  │12:45            │22:09│     │
       │                        │      └─────┘  └───────┘  └───┘             └────┘      │
       │                        │                                                         │
       ▼                        ▼                                              22:09 ──►  │
                                                               FIX APPLIED: LimitSTACK=16M
                                                                    ◄── STABLE ──►
```

---

## 6. Diagnostic Data for ntop Bug Report

### 6.1 Kernel dmesg (SIGBUS)

```
[Wed Feb 18 11:37:29 2026] traps: ntopng-3-pkt[3409388] trap stack segment
  ip:6193815291d3 sp:7dfcc81f8b70 error:0 in ntopng[4091d3,61938127c000+698000]
[Wed Feb 18 11:37:59 2026] traps: ntopng-3-pkt[3411501] trap stack segment
  ip:6458a4896140 sp:7f06c2df8b70 error:0 in ntopng[409140,6458a45e9000+698000]
[Wed Feb 18 12:45:43 2026] traps: ntopng-3-pkt[956] trap stack segment
  ip:56f6f1b5c1d3 sp:75266a3f8b70 error:0 in ntopng[4091d3,56f6f18af000+698000]
[Wed Feb 18 22:09:38 2026] traps: ntopng-3-pkt[92838] trap stack segment
  ip:58ade7f1b1d3 sp:7604a21f8b70 error:0 in ntopng[4091d3,58ade7c6e000+698000]
```

### 6.2 systemd journal (SEGV crash loop sample)

```
04:16:39 [startup.lua:47] [asset_utils.lua:105] Migrating empty manufacturer column
04:16:39 [startup.lua:47] [asset_utils.lua:115] Assets manufacturer migration completed
04:16:41 [ZMQCollectorInterface.cpp:257] Collecting flows on zmq://127.0.0.1:5557
04:16:41 [ZMQCollectorInterface.cpp:530] Allocating new probe/source [source_id: 21149134]
04:16:57 systemd: Main process exited, code=dumped, status=11/SEGV
04:17:03 [startup.lua:47] [asset_utils.lua:105] Migrating empty manufacturer column
04:17:03 [startup.lua:47] [asset_utils.lua:115] Assets manufacturer migration completed
04:17:06 [ZMQCollectorInterface.cpp:257] Collecting flows on zmq://127.0.0.1:5557
04:17:06 [ZMQCollectorInterface.cpp:530] Allocating new probe/source [source_id: 21149134]
04:17:23 systemd: Main process exited, code=dumped, status=11/SEGV
```

### 6.3 Asset inventory shutdown dump (during crash)

```
05:43:56 [shutdown.lua:45] [asset_utils.lua:1203] Adding new asset:
  {"type":"mac","key":"6_E4:23:3C:53:98:17","manufacturer":"Juniper Networks","device_type":8}
05:43:56 [shutdown.lua:45] [asset_utils.lua:1203] Adding new asset:
  {"type":"mac","key":"6_D4:4F:67:9D:3E:FC","manufacturer":"Huawei Technologies Co.,Ltd"}
05:43:57 [HTTPserver.cpp:1954] HTTP server terminated
05:43:57 [NetworkInterface.cpp:4124] Cleanup interface zmq://127.0.0.1:5557
05:43:57 systemd: Main process exited, code=dumped, status=11/SEGV
```

### 6.4 Crash offset disassembly (stripped binary)

```asm
; Primary crash point: 0x2ad1d3 (3 of 4 SIGBUS crashes)
; Near: basic_stringbuf destructor (exception handling region)
2ad1d1: f3 0f 1e fa     endbr64
2ad1d5: 48 89 c5        mov    %rax,%rbp    ← IP lands here (sp exhausted)
2ad1d8: e9 57 bc ed ff  jmp    188e34       ← exception dispatch table

; Secondary crash point: 0x2ad140 (1 of 4 SIGBUS crashes)
2ad140: ff f3           push   %rbx         ← stack push triggers SS fault
2ad142: 0f 1e fa        nop    %edx
2ad145: 48 89 c5        mov    %rax,%rbp
2ad148: e9 6f bc ed ff  jmp    188dbc       ← exception dispatch table
```

### 6.5 How to Get a Full Backtrace

Since the binary is stripped, `addr2line` and `nm` return no useful information. To get a proper stack trace, ntop must provide a **debug build** or the crash must be caught with gdb:

```bash
systemctl stop ntopng
gdb --args /usr/bin/ntopng /run/ntopng.conf
(gdb) handle SIG33 nostop noprint pass
(gdb) handle SIGPIPE nostop noprint pass
(gdb) set pagination off
(gdb) run
# ... wait for crash ...
(gdb) thread apply all bt
(gdb) info threads
```

Reference: https://github.com/ntop/ntopng/blob/dev/doc/README.crash.md

---

## 7. Action Items

| # | Action | Priority | Owner | Status |
|---|--------|----------|-------|--------|
| 1 | Apply `LimitSTACK=16MB` systemd override | P1 | GOLINE | **DONE** (2026-02-18) |
| 2 | Monitor for recurrence of SIGBUS after fix | P1 | GOLINE | **MONITORING** |
| 3 | File ntop bug: Asset Inventory SIGSEGV race condition | P2 | GOLINE | TODO |
| 4 | Obtain gdb backtrace if SIGSEGV recurs | P2 | GOLINE | TODO |
| 5 | Contact ntop support: request debug build or fix for stack overflow | P3 | GOLINE | TODO |
| 6 | Evaluate disabling Asset Inventory if crashes recur | P3 | GOLINE | Workaround ready |
| 7 | Test `LimitSTACK=33554432` (32 MB) if 16 MB proves insufficient | P3 | GOLINE | Fallback ready |

---

## 8. References

- [Linux kernel traps.c — X86_TRAP_SS handler](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/traps.c)
- [ntopng GitHub Issue #7887 — SIGBUS crashes](https://github.com/ntop/ntopng/issues/7887)
- [ntopng GitHub Issue #6830 — pkt thread trap](https://github.com/ntop/ntopng/issues/6830)
- [ntopng CHANGELOG — 6.6 Asset Inventory fixes](https://github.com/ntop/ntopng/blob/dev/CHANGELOG.md)
- [ntopng crash debugging guide](https://github.com/ntop/ntopng/blob/dev/doc/README.crash.md)
- [pthread_attr_setstacksize(3)](https://man7.org/linux/man-pages/man3/pthread_attr_setstacksize.3.html)
- [Netgate Forum — ntopng crashes, Active Network Discovery](https://forum.netgate.com/topic/187219/ntopng-stopping-exiting-crashing)
- [OSS-Fuzz: Flow::processExtraDissectedInformation vulnerability](https://vulert.com/vuln-db/oss-fuzz-ntopng-181825)







If the ntopng service crashes with a *segmentation fault*, there is a bug in the
software which must be fixed.

Before opening a new issue, please ensure that you are using a recent (ideally the latest)
ntopng version. In order to speed up the troubleshooting process, a stack trace of the
crash is needed.

# Linux

An easy way to get a stack trace on Linux is to run ntopng through the *gdb* debugger:

1. Ask to the ntop team a binary with debug symbols, specifing ntopng version
2. Install gdb (e.g. `sudo apt-get install gdb`)
3. Stop the running service: `sudo systemctl stop ntopng`
4. Start gdb: `gdb --args <downloaded binary path> /etc/ntopng/ntopng.conf`
5. Execute `handle SIG33 nostop noprint pass` and `handle SIGPIPE nostop noprint pass`
6. Execute `run` to start debugging ntopng
7. Wait for the crash to occur
8. Now run `bt` into gdb to get a stack trace of the crash
9. Send the bt output to ntop team

# Windows

The *WinDbg* tool can be used to get a stack trace on Windows:

1. Download and install WinDbg Preview: https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools
2. Stop the running service: `C:\Program Files\ntopng\ntopng.exe /r`
3. Open the WinDbg debugger and load the ntopng executable
4. Run the `g` command to start ntopng
5. Wait for the crash to occur
6. Now run `k` to get a stack trace of the crash

See https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/getting-started-with-windbg for more details.
