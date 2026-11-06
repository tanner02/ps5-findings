# PS5 Findings

**Contents / legend:** [Results](#results) · [Tested hardware](#tested-hardware) · [Astro Bot FSR](#astro-bot-fsr) · [Astro Bot PSSR](#astro-bot-pssr) · [PS4 image enhancement](#ps4-image-enhancement) · [SISR startup checks](#sisr-startup-checks) · [SISR shader package](#sisr-shader-package) · [Native integration lessons](#native-integration-lessons) · [Remaining gaps](#remaining-gaps) · [References](#references)

Reverse-engineering and live-test findings, recorded **6 November 2026**. **Verified** means directly measured or checked in the inspected binaries. **Inference** and **unverified** identify the limits of that evidence.

## Results

| Feature | Finding on the tested base PS5 |
|---|---|
| Astro Bot FSR 3 temporal upscaling | **Verified:** 1920×1080 input → 3840×2160 output, including native standalone ELF execution. |
| Astro Bot PSSR | **No working unlock.** Game, PSML and AGC checks are separate; earlier experiments caused initialization failures and a reported kernel panic. |
| Sony PS4 enhancement preference | **Verified:** the actual setting can be enabled and restored by a payload. |
| Sony PS4 enhancement processing | **Not activated:** Driveclub did not load the SISR manager or allocate the compositor's processing resources with the preference enabled. |

FSR temporal upscaling, PSSR and the PS4 SISR path are distinct implementations. The FSR result does not establish either Sony neural path working on base hardware. No frame generation was implemented.

## Tested hardware

- Base PS5, system software **12.00**, raw firmware value **`0x12000043`**.
- Astro Bot **`PPSA21567`**, version **`01.019.000`**.
- Driveclub **`CUSA00003`** for the PS4 enhancement test.

Offsets below are **module-relative RVAs for the inspected builds**, not reusable absolute addresses. Other firmware, regions and game versions require independent validation.

## Astro Bot FSR

### Rendering integration

A custom FSR 3 temporal-upscaling backend was integrated at Astro's existing **TAA/TaaLite** stage. It uses the game's color, depth, motion and jitter inputs and writes to its output/history destination. No game files were replaced.

- The successful profile keeps 3D input, depth and motion at **1920×1080**, while output/history remain **3840×2160**.
- Lowering internal resolution alone initially produced **2432×1368** output/history. A separate post/history setting was needed for the tested 4K destination.
- **TAA must remain enabled:** this integration depends on its existing render path.
- This hook does not establish an implementation where all post-processing stays at 1080p and only the final image is enlarged. Moving the upscale later would require another destination and preservation of the temporal inputs.

### Jitter and image quality

Normal temporal history with the original jitter produced vertical-line artifacts. Resetting history each frame removed the lines. Retaining history while disabling jitter reduced the lines but left blockiness, so that diagnostic was not accepted as a fix.

The best tested configuration expanded the native **CL0 current and previous jitter by 4×**. Four helper instructions use existing **960/540** divisors instead of **3840/2160**; CL1 remains unchanged. Changing the native helper keeps projection jitter and the temporal input consistent. Scaling only the offset supplied to FSR would not establish that consistency.

The resulting configuration retained normal history and improved the image substantially. Mild distant-detail blockiness remained. Similar, softer artifacts were also observed with native TAA under the same XML configuration; their exact origin was not established.

**Cleanup restores native TAA, but does not undo the XML's resolution or expanded-jitter changes.** A stock comparison requires disabling that XML profile and fully relaunching the game.

### Verified execution and limits

- Expanded-jitter gameplay: **8,833 submitted and retired frames**, 8,832 temporal frames and one reset, with no reported runtime errors and complete cleanup.
- Standalone resident ELF: **3,242 submitted and retired frames**, 3,241 temporal frames and one reset, with zero reported busy/invalid/fatal errors.
- The resident worker continued after its sender disconnected. A second TCP launch requested a clean stop; native TAA and GP/FP state were restored and owned allocations were released.
- Import into **itsPLK Payload Manager 0.5.2**, stored bytes and manager listing were verified. **Manual UI launch and automatic activation across a fresh game launch/relaunch were not confirmed.**
- No controlled performance benchmark or artifact-free image-quality result is claimed.

The tested resident ELF is **2,451,648 bytes**, SHA-256:

```text
68db5d74f30ab74fa9661b06b54c1507bd7898eb56a4ce9b9e6d01d4df626994
```

It requires PS5Debug and the **“1080p FSR 3 with Expanded Jitter (Experimental)”** XML profile. Its runtime connections are console-local, so it does not require a connected PC. itsPLK's manager and etaHEN's raw-ELF menu have different relaunch behavior: etaHEN can terminate the previously tracked payload process, making the same resident-worker arrangement unsuitable without a separate frontend.

### Useful exact-build RVAs

| Item | RVA |
|---|---|
| TaaLite entry; original prologue `55 48 89 e5` | `0x7048BC0` |
| Native handoff hook; original bytes `48 8b 05 fb b9 e6 07` | `0x704931E` |
| Native handoff continuation | `0x7049325` |
| Force-1080 selector | `0xC62A85` |
| Full post/history setting | `0x7385005` |
| CL0 current/previous jitter instructions | `0x734A626`, `0x734A6BA`, `0x7357D66`, `0x7357DFA` |

## Astro Bot PSSR

The inspected game contains a real PSSR integration: `Pipeline_Pssr_Enable`, PSSR render stages, velocity rewriting and PSSR history/output resources. Its metadata already includes Trinity-support bit 22 (`attribute3 = 0x440014`) and `psml.mfsrVersion = 09.40`. **Support metadata does not force hardware mode.**

The game's PSSR initialization is gated by a graphics-object flag at **`object + 0x308ADF`**. The positive path loads system module **`0x137`** and calls PSML initialization. Separately, the startup hardware-query result influences the game's memory budget by **`0x18800000`**; globally spoofing mode has effects beyond selecting PSSR.

Measured firmware-12 native results:

| Operation | Result |
|---|---|
| Load PSML system module `0x137` | Success |
| PSML initialization on the base console | `0x8A810012` |
| Initialization after temporarily changing PSML's cached Trinity byte | `0x8A810011` |

The cached byte was restored. A kernel panic was subsequently reported during the experimental sequence; **the precise cause was not established**. No successful PSSR output was demonstrated.

Offline inspection found another independent restriction: AGC's shader-creation path rejects the inspected type-`0xF` shaders with **`0x8A6C003D`** unless its own capability byte is enabled. PSML creates both type-`0xE` and type-`0xF` groups. This is evidence of a driver-side gate, **not a measured explanation of the earlier initialization failure**. A repeated first-shader result of `0x8A6C001F` indicates an already-initialized header and is not proof of ISA incompatibility.

No game flag, metadata bit or cached mode-byte change demonstrated a working PSSR backend on base hardware.

## PS4 image enhancement

Sony markets **“Enhance Image Quality for PS4 Games”** for PS5 Pro. The firmware contains a PS4 SISR subsystem; its connection to that feature is strongly supported by the setting, module names and launch/compositor behavior.

### Actual setting and live result

| Setting | Meaning |
|---|---|
| RegMgr `0x0A300000`, `VIDEOOUT_sisr_enable` | Actual user preference; enable and restore were verified by native read-back. |
| RegMgr `0x7802FB01`, `BC_SISR_ENABLE` | Separate QA/debug override with additional readiness checks; not the normal preference. |
| `machdep.rcmgr_bc_sisr` | Read-only integer; returned **1** on this base console. Its full meaning is unresolved and it is not a writable enable switch. |

The preference was enabled **before launching Driveclub**. The game mapped neither the SISR manager nor the BC neural module. The compositor had **no SISR shared buffer, valid-DMEM flag 0, owner PID 0 and no BC neural module**. The original preference value was restored afterward.

This directly establishes that **changing the setting alone did not activate processing in that test**. It does not prove every conceivable base-console implementation impossible.

### Components

| Component | Observed role |
|---|---|
| ShellCore `ps4_sisr_ctl` | Per-title deny policy and `mount_param.ps4SisrDenied`; references `/system/priv/ps4sisrctl_ro/ps4sisrctl.txt`. |
| `libSceSisrSys.sprx` | Compositor-side service, loaded even on the base console. Presence alone does not mean processing is active. |
| `libSceSisrMgr.sprx`, internal `0x800000D5` | Game-side manager and VideoOut postprocessor integration. |
| `libScePsmlBcSisr.sprx`, internal `0x800000D7` | BC neural processing implementation. |

Both internal modules loaded in an isolated diagnostic payload. The BC CPU-only requirements query succeeded and returned **308,918,684 bytes**, aligned to **2,097,152 bytes**. This was a queried requirement, not an allocation made by the diagnostic.

## SISR startup checks

The manager's own first initialization check is **`sceKernelIsPs4Process`**, not a Pro-mode check. Its constructor invokes that initialization automatically. Further checks include application/display state; the inspected path requires output resolution above 1080p.

Stock Sysmodule startup has a separate hardware gate:

- Imported NID **`yu17wG8L5FI`** queries `hw.sce_main_socid` and returns true only on a successful query with **`(socid & 0xFFFFFFE0) == 0x840FC0`**. Its exact symbol name remains unresolved.
- Sysmodule calls it at RVAs **`0x32EC`** and **`0x39D1`** and requires a result of **1**. The positive path selects table index **`0x2A`**, verified to identify `0x800000D5` / `libSceSisrMgr`.
- Other process-mode, registry and state checks also apply. One registry gate uses hashed key **`0x339B6083A3DD4AB1`**; it has not been equated with the normal user-preference key.
- ShellCore's flag initialized from this SoC predicate was **0**, including after a power cycle.

**Do not confuse this predicate with `sceKernelIsTrinityMode`.** The latter has NID **`tU5e3f9gSiU`** and independently returned **0**. BC initialization contains an explicit Trinity check and a “Current mode isn't Trinity” error path.

A full BC-init diagnostic stopped in shared-runtime bootstrap before returning. It established neither successful initialization nor a returned hardware error. No SISR GPU dispatch was performed.

## SISR shader package

The console contained `/system/common/psml_package/BCSISR_v070.psp`:

| Property | Verified value |
|---|---|
| Package size | 5,616,272 bytes |
| SHA-256 | `55ba880f6d752f562422febad3a9669ef93a0f0377b6dc6257fdcd65c650d13d` |
| Profiles | `1080p`, `2160p` |
| Shader blob per profile | 2,776,848 bytes |
| Footer-defined executable size per profile | 2,657,048 bytes; 2,657,056 after 16-byte alignment |
| Compiler target / literal shader-class byte | **12** / **`0x00`** |
| Separate profile data | 30,336 bytes per profile; its complete interpretation is unresolved |
| Trailing debug region | 119,744 bytes per profile, beginning `sl00`, before the `barefoot` footer |

The referenced alternate directory `BCSISR_unetv1` was absent on the tested firmware.

The debug data contains 2,722 label records and 152 unique strings describing convolution, downshuffle/upshuffle, detile/retile and other stages. The recorded offsets are **string positions, not verified instruction PCs**. No validated SL00 schema or complete model graph was recovered.

Recurring patterns occur inside the footer-defined executable range, but **byte-pattern counts are not instruction disassembly or proof of incompatibility**. The examined decoder sources do not provide semantics for the relevant family-`0x39` forms, SMEM opcode `0x2B`, or SOPP opcodes `0x2A`/`0x2B`. Some recognize instruction width before rejecting the opcode; that is not a translation backend.

Source revisions checked: KytyPS5 **`182161eb3e31439d97433607a8aa95463c56a9cc`** and AnyPS5 **`b96af01bebe2d549567fba4d6c0d4ca2fed19922`**. The additional Trinity support examined there concerns BVH/ray tracing, not the missing neural operations.

## Native integration lessons

- Use a GP/FP-preserving native-call bridge with a private stack and xstate storage. Raw PS5Debug `SYS_PROC_CALL` was not a reliable substitute in the tested render-thread integration.
- PS5Debug can expose the already-executed `PUSH RBP` at TaaLite entry+1. The tested bridge validates the saved RBP, emulates `MOV RBP,RSP`, and resumes at entry+4.
- Kernel-assisted dynlib symbol lookup resolved system-library exports that normal `sceKernelDlsym` did not resolve in the diagnostic context.
- Direct CPU reads of executable PRX pages faulted despite apparently readable map entries. `kernel_proc_copyout` into a local buffer worked for signature checks.
- Memory maps contain overlapping coarse/auxiliary entries. Validate effective permissions across boundaries rather than trusting a broad RW entry.
- Hook removal is not sufficient cleanup: wait for active guards and GPU retirement before destroying context/resources, and release the CPU module only after no thread is executing it. Rediscover processes and revalidate signatures after every game restart.

## Remaining gaps

There is **no verified functional PSSR or PS4 SISR unlock for the tested base PS5**. A working base-compatible implementation still needs validated semantics for the proprietary neural operations, or a validated portable implementation of the same processing. Bypassing loader checks and enabling preferences do not supply that implementation.

The Astro FSR result is narrower but demonstrated: temporal upscaling for the exact tested game build/profile, with the image-quality and launcher-validation limits described above.

## References

- [Sony: Enhance Image Quality for PS4 Games](https://www.playstation.com/en-ie/support/games/enhance-games-ps5-pro/) — official feature scope.
- [AMD FidelityFX SDK](https://github.com/GPUOpen-LibrariesAndSDKs/FidelityFX-SDK) — FSR implementation reference.
- [KytyPS5 decoder at the reviewed revision](https://github.com/KytyPS5/KytyPS5/tree/182161eb3e31439d97433607a8aa95463c56a9cc/src/graphics/shader/recompiler/frontend/decode) — reviewed decoder coverage.
- [AnyPS5 shader sources at the reviewed revision](https://github.com/boykopovar/AnyPS5/tree/b96af01bebe2d549567fba4d6c0d4ca2fed19922/core/shader) — reviewed decoder coverage.
- [PS5 payload SDK](https://github.com/ps5-payload-dev/sdk), [itsPLK Payload Manager](https://github.com/itsPLK/ps5-payload-manager), [etaHEN](https://github.com/etaHEN/etaHEN) — native payload and launcher references.

This repository documents findings. It does not include Sony firmware, game binaries, captured neural packages or a downloadable unlock payload.
