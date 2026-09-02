# A QEMU core - analysis and plan

Mainline QEMU as a Chimera waterbox core, after the xemu port. xemu IS a QEMU
(10.2.0 base) already proven inside miniBox, so this port is mostly *the same
mechanisms pointed at a different machine*: instead of an Xbox it is a PC
(`pc` i440fx, TCG, one vCPU, icount), and instead of copyrighted console
firmware it boots free software end to end - SeaBIOS bundled as a package
asset, a small Linux distro as the disc. The goal for now: **boot a small
Linux distro to a shell inside the Chimera GUI**, deterministically, with
savestates and movies.

- New repository `chimera-core-qemu` (mirrors `chimera-core-xemu`), submodule
  at `extern/cores/qemu`, nested `extern/qemu` pinned to the same QEMU
  release family xemu rides (10.2.x) so the xemu patches port near-verbatim.
- The frontend's share is one line (`PC` in SystemNames); everything else
  rides the package, like xemu.

## Decisions up front

| Question | Answer | Why |
|---|---|---|
| Target binary | `qemu-system-x86_64` (x86_64-softmmu) | modern small distros (Alpine, TinyCore) are x86_64; the port surface (accel/tcg, target/i386 SSE helpers, block, main loop) is the same one xemu proved on i386-softmmu |
| Machine | `pc` (i440fx), 1 vCPU, fixed `-cpu` (no `rdrand`/`rdseed`), `-icount shift=0,sleep=off`, `-rtc base=2000-01-01,clock=vm` | the xemu argument list, verbatim philosophy; SMP>1 under green threads is scheduling-nondeterminism for nothing |
| ROM slot | the distro ISO (El Torito), `min 1` | one file = the wizard story; direct `-kernel` boot stays available as a gate leg |
| Firmware | none from the user - SeaBIOS (`bios-256k.bin`, vgabios, `linuxboot_dma.bin`) rides the package **assets channel** (82aef36), mounted by name at Init | SeaBIOS is LGPL-3.0: the first Chimera machine whose boot firmware can ship in the package. `package-licenses.json` carries the terms |
| Video | VGA scanout: per-frame `graphic_hw_update(console 0)` → pixman → BGRA | xemu M2's mechanism; text mode included (ui/console renders the glyphs), no GL bridge needed - a distro console is CPU-drawn |
| Input | PS/2 keyboard: declared buttons in qcode order; keys sent as `qemu_input_event_send_key_qcode` press/release diffs at the frame boundary, machine paused | the device pushes (it does not poll), and the boundary is a pure virtual instant, so delivery order is deterministic; 100+ keys ride `SetButton`, a packed word is useless here (64-bit) |
| Audio | deferred (AC97/SB16 need the xemu-APU-style virtual-clock treatment) | not needed to boot a distro; `audio.get` omitted until then |
| systemId | `PC` -> "IBM PC (QEMU)" | no collision in SystemNames; `.iso` already routes to XBOX by first-declared (db0a53f), so the wizard's system picker is the entry |
| Frame quantum | VGA 60 Hz: `VBLANK_NS 16666667`, vsync `1000000000/16666667` | a PC under `-display none` has no refresh timer either - the driver's one-shot virtual timer IS the vblank, exactly xemu's model |
| Memory layout | start from xemu's `[512, 16, 128, 256, 2048]`, prove and tune with run-wbx | mainline QEMU minus the Xbox devices is slimmer, but machine RAM (say 256MB) rides the mmap arena like xbox.ram did |

## What transfers, what is new

The xemu port splits cleanly into "core QEMU machinery" (ports near-verbatim)
and "Xbox devices" (not needed at all). Patch by patch:

- **0006 canonical state** (migration/savevm.c, system/cpu-timers.c: host-clock
  deltas canonicalised) - ports verbatim. Also unlocks the gate: a dumpable,
  comparable migration stream.
- **0007 the TLS wall** (coroutine-tls over pthread keys, `current_cpu` behind
  an accessor, `-D__thread=` sweep, PIXMAN_NO_TLS) - ports verbatim; mainline's
  TLS symbol set is close to xemu's 26. `readelf -sW core.wbx | awk '$4=="TLS"'`
  stays empty, check-wbx enforces.
- **0008 warp only on halt + ptimer respin** - the `accel/tcg` half (a merely
  STOPPED cpu must not count as idle) ports verbatim; the NV2A PTIMER half is
  Xbox-only.
- **0010 MAP_STACK** (util/oslib-posix.c: QEMU's stack allocations must say so)
  - verbatim.
- **0001-0003 headless build** - mostly unnecessary: mainline builds headless
  stock (`--disable-sdl`, `-display none`). Only the disk-latency wiring of
  0003 and small `vl.c`/net tweaks carry ideas over.
- **0004 (APU virtual clock), 0005 (pfifo), 0009 (GL bridge)** - Xbox devices;
  not needed for a VGA-console PC.
- Driver sources: `guest-ucontext.c` (70-line makecontext/swapcontext),
  `guest-syscalls.c` (fake eventfd/pipe/signalfd, yielding ppoll, preadv/pwrite,
  deterministic getrandom, sigsetjmp alias), `chimera-latency.c` (every block
  request completes at the NEXT FRAME BOUNDARY, writes to an in-memory COW
  overlay) - these are target-agnostic QEMU machinery: copy, then adjust.
  `det-pow.c`, `samplerate/`, `monitor-null.c`, the whole GL stack: stay home.
- The driver itself (`qemu-waterbox.c`): the frame loop, warp governor,
  frame_boundary vm_stop dance, `Init/FrameAdvance/GetStateSize/GetStateData/
  GetLoadError`, memory domains (`pc.ram` is the real block on a PC - no decoy),
  video via the console surface - all lift from `xemu-waterbox.c` with the Xbox
  input/UI hooks deleted and the PC argument list in their place.

New work, none of it novel:

1. **The input push model.** xemu's pads are polled state; the PC's i8042 is
   event-driven. FrameAdvance diffs the declared key set and emits key events
   while paused - `qemu_input_event_send_key_qcode` per changed key, scancode
   set 2 through ps2. Keyboard LEDs and the ctrl-alt-shift mega-keys are the
   only subtleties; the gate holds a key down and demands the machine diverge
   from the plain run exactly while held (xemu's input leg, same shape).
2. **Guest-visible randomness.** `xemu` never hit this: Xbox software does not
   `rdrand`, Linux does. QEMU's TCG `rdrand` helper draws host randomness -
   guest-visible nondeterminism, breaking native A == B at the first kernel
   entropy touch. Two layers: (a) don't expose `rdrand`/`rdseed` in the `-cpu`
   model; (b) if the kernel then starves CRNG init (boot hangs on `getrandom`),
   patch the helper under `CHIMERA_GUEST` to a fixed-seed PRNG - the guest-side
   twin of `guest-syscalls.c`'s deterministic getrandom.
3. **MXCSR at PC scale.** xemu's lesson 16 applies *more* here: x86-64
   userspace sets MXCSR constantly (glibc SSE), and TCG's ldmxcsr helper
   executes on the host, so the emulator's own float math inherits guest
   rounding on any thread that ran guest code - in the sandbox that is every
   green thread. Audit QEMU's own float paths (timekeeping, block math) and pin
   default MXCSR around them, the way `apu_vframe_tick` did.
4. **SeaBIOS + distro boot determinism audit.** Everything SeaBIOS reads is
   fw_cfg or the emulated RTC (virtual, base pinned); SMBIOS/UUID defaults are
   fixed when unspecified. Watch: PIT/HPET periodic timers advancing by
   ABSOLUTE deadline (xemu mechanism 8), and the El Torito reads completing at
   frame boundaries through the block filter.

## Milestones

- **M0 - deps and skeleton.** Repo, submodules, `setup-guest.sh`/
  `setup-native.sh`/`build-deps.sh` copied from xemu (zlib, glib + pcre2/
  libffi wraps, pixman - SHA-pinned, static musl). Pin `extern/qemu` at the
  10.2.x release matching xemu's base; apply-patches.sh starts its numbered
  series with the four verbatim ports.
  Gate: guest `qemu-system-x86_64` links, `check-wbx.sh` clean.
  **DONE (53780b1).** Seven-patch series (four verbatim ports plus the
  pixman-shareable, ATA-SRST, QAPI-latency and driver-build patches), both
  builds green, `check-wbx.sh` clean. Guest-deps wrinkle worth remembering:
  the host's `/usr/include/linux` must be overwritten by QEMU's bundled
  `linux-headers/` copies or the guest build breaks on iommufd.
- **M1 - headless boot, native==sandbox.** Driver: frame loop, governor,
  boundary, migration-stream exports. Boot SeaBIOS (assets) with no disc to
  the "no bootable device" screen, then `-kernel` a pinned buildroot/busybox
  image as the simplest Linux leg. `run-gate.sh N`: native A == B ==
  sandbox, byte for byte, at 60 and 600 frames.
  **DONE IN SUBSTANCE at 120 frames (chimera-core-qemu 5c78986).** Native
  A == B == sandbox, byte for byte, state stream and video, savestates
  exact, with the Alpine ISO attached from frame 0 (SeaBIOS through El
  Torito - i.e. the M2 disc path too). Two determinism bugs found and fixed
  on the way: timer-offset canonicalisation needed the `CHIMERA_WATERBOX`
  guard (patch 0002), and ATA SRST completion had to become synchronous
  (patch 0007) - found by TB-trace bisection at the first divergent guest
  instruction. Not yet done to the letter: the 600-frame (and longer) gate
  points, and the separate no-disc / `-kernel` legs (skipped in favour of
  the real ISO, which exercises strictly more).
- **M2 - the distro.** ISO slot + El Torito through the latency filter, PS/2
  input, `pc.ram` domain, savestate rerecord around every frame, arena
  snapshot byte-identical. Gate: Alpine (or TinyCore) ISO boots to a busybox
  shell; a held key changes the machine exactly while held; native ==
  sandbox under the press.
  **IN PROGRESS.** Everything through the ISO slot is in place and gated at
  120 frames; the open work is the long run (thousands of frames, all of
  Linux boot) native == sandbox, the input leg, and observing the boot to a
  shell. First long-run attempt found the sandbox binary was stale (built
  before the QAPI `cow` option; rejected `-blockdev ... cow=on`) - rebuilt;
  the long run itself was interrupted before concluding.
- **M3 - a real core package.** `waterbox.config` (video buffer cap
  1280x1024 for VESA modes, 60 Hz vsync from `VBLANK_NS`), `file_slots.json`
  (the disc), keybinds, licences (GPL-2.0 QEMU + LGPL SeaBIOS), build.json
  provenance, `SystemNames` line. `run-frontend.sh`: the GUI boots the distro
  headless (Xvfb), the frontend machine's `pc.ram` slice equals run-wbx's,
  and a typed key lands in the guest.

Deferred beyond the goal: audio, mouse/tablet, persistent disk export (the
COW overlay persists through savestates/movies but is not yet a file), a
q35/`-machine` setting, i386 distros.

## Frontend share (the whole of it)

- `SystemNames.cs`: `["PC"] = "IBM PC (QEMU)"`.
- The submodule entry in `.gitmodules`.
- Nothing else: the RomGame SHA1 streaming and optional-firmware handling
  xemu needed are already home; the assets channel is already home.

## Open questions, with a default each

- **Which distro is the gate's reference disc?** Default: Alpine x86_64
  standard ISO (small, boots to login on the VGA console); a buildroot image
  is the deterministic-minimal alternative if Alpine's userspace entropy
  (walltime reads are fine - they are virtual) or boot length misbehaves.
- **Shift.** icount `shift=0` (a ~1GHz machine) boots Linux in minutes of
  wall clock; if boot-to-shell exceeds ~10k virtual frames, revisit (the
  frame budget is a settings-time constant, and movies cite it).
- **`-cpu` model.** Default: `qemu64` + explicit feature additions minus
  `rdrand`/`rdseed`; pin exactly, because the CPUID set is machine identity.
