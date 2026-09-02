# A QEMU core - status and plan

Mainline QEMU as a Chimera waterbox core, after the xemu port. xemu IS a QEMU
(10.2.0 base) already proven inside miniBox, so this port is mostly *the same
mechanisms pointed at a different machine*: instead of an Xbox it is a PC
(`pc` i440fx, TCG, one vCPU, icount), and instead of copyrighted console
firmware it boots free software end to end - SeaBIOS bundled as a package
asset, a small Linux distro as the disc.

**The original goal - boot a small Linux distro to a shell inside the Chimera
GUI, deterministically, with savestates and movies - is MET.** TinyCore
CorePure64 17.1 boots to `tc@box:~$` through the real frontend, and the
frontend's machine is byte-identical to the bare sandbox's at every layer
checked. The disc path is now fast too: **disc boot to a live shell is ~2
minutes wall** (was 15-25). What remains is a full distro running real
software.

- Core repository `chimera-core-qemu` (mirrors `chimera-core-xemu`), submodule
  at `extern/cores/qemu`, nested `extern/qemu` pinned to the same QEMU release
  family xemu rides (10.2.x).
- The frontend's share is one line (`PC` in SystemNames); everything else
  rides the package, like xemu.

## Machine identity (decided, movies cite it)

| Knob | Value |
|---|---|
| Machine / CPU | `pc` (i440fx), 1 vCPU, TCG, `-cpu qemu64` (no `rdrand`/`rdseed`) |
| icount | `shift=0` (a 1GHz-class virtual CPU, 1ns/instruction) - 4x the compute budget per virtual second; env override `CHIMERA_ICOUNT_SHIFT` (native test builds) |
| Governor | period 1ms (`GOVERNOR_NS 1000000`) |
| Boot order | `-boot order=d` (don't walk floppy/hard disk first) |
| Frame quantum | VGA 60 Hz: `VBLANK_NS 16666667`, vsync `1000000000/16666667` |
| RTC | `base=2000-01-01,clock=vm` |
| RAM | 256 MiB default (`ramMiB` setting: 128/256/512) |
| Memory layout | `[512, 16, 128, 256, 2048]` MiB (xemu's numbers, proven at this scale) |
| Video | VGA scanout per frame → pixman → BGRA; text mode included |
| Input | PS/2 keyboard, buttons in QKeyCode order, press/release diffs at the paused frame boundary |
| Firmware | SeaBIOS + VGABIOS + linuxboot_dma ride the package **assets channel** (LGPL, shippable) |
| Disc timing | async IDE completions deliver at (last IDE port access + 0ns virtual, i.e. effectively synchronous; `CHIMERA_IDE_LATENCY_NS 0`); sync ATAPI completes in the port access itself (patch 0010) |

## What transfers, what is new (from the xemu port - kept for reference)

- **Ported near-verbatim:** canonical state (migration/savevm host-clock
  deltas), the TLS wall (coroutine-tls over pthread keys, `-D__thread=` sweep,
  PIXMAN_NO_TLS), warp-only-on-halt, MAP_STACK, the headless build ideas.
- **Copied and adjusted:** `guest-ucontext.c`, `guest-syscalls.c` (yielding
  ppoll, deterministic getrandom, fake eventfd/pipe/signalfd),
  `chimera-latency.c` (block requests complete at the NEXT FRAME BOUNDARY,
  writes to an in-memory COW overlay), the `qemu-waterbox.c` driver itself.
- **Xbox-only, stayed home:** APU virtual clock, pfifo, the GL bridge,
  NV2A anything.
- **New for the PC:** the input push model (i8042 is event-driven; keys are
  `qemu_input_event_send_key_qcode` diffs pushed after `vm_start` - pushing
  while paused silently drops events), no-`rdrand` CPUID, direct kernel boot
  (below), and the QEMU-side determinism fixes found by the gate:
  - *ram_save_iterate MAX_WAIT*: the save path broke out of the RAM-section
    loop on a 50ms host wall-clock threshold, splitting the section at
    host-speed-dependent points (native and sandbox framed the same bytes
    differently). Patch 0008: the bail-out is compiled out under
    `CHIMERA_WATERBOX`.
  - *ATA SRST completion* had to become synchronous (patch 0007).
  - *The initrd is read, not mapped* (patch 0009): `mmap` of a mounted file is
    not a thing in the sandbox VFS.

## Two boot legs, two file conventions

1. **Direct kernel boot (the fast leg).** The project fills `kernel` and
   `initrd` slots; `linuxboot_dma` loads them over fw_cfg DMA - no firmware
   disc traffic at all. TinyCore reaches its shell in 3000 frames (~30s
   wall). This is the default project (`~/chimera/projects/tinycore-direct`)
   and the iteration path.
2. **Disc boot (El Torito through SeaBIOS).** The project fills the `disc`
   slot. Deterministic, gated, and fast: ISOLINUX's 30s `boot:` timeout
   fires at frame ~1800 (30 virtual seconds), the 26 MB kernel+initrd load
   completes in a few hundred more frames, and a live shell answers keys by
   ~frame 4600 - about 1m50s wall end to end. A movie (or the GUI user) can
   press Enter at the prompt to skip the 30 virtual-second wait.

File-name conventions the driver understands (it parses the mounted `slots`
map at Init, falling back to the flat names): project files mount under their
canonical names (`vmlinuz64`), and the engine additionally mounts the first
non-support file as `rom`/`rom.name` - the driver treats a kernel-slot primary
as a direct boot, not as a CD, and a project without a disc gets an empty
drive (an empty drive is a valid machine: SeaBIOS reports no bootable device).

## Milestones - all done

- **M0 - deps and skeleton - DONE (53780b1).** Seven-patch series, both builds
  green, `check-wbx.sh` clean. Remember: the host's `/usr/include/linux` must
  be overwritten by QEMU's bundled `linux-headers/` or the guest build breaks
  on iommufd.
- **M1 - headless boot, native == sandbox - DONE (5c78986).** Byte-for-byte
  state stream and video, savestates exact, found by TB-trace bisection at the
  first divergent guest instruction.
- **M2 - boot to shell - GATED (314ecd1).** Direct kernel boot: TinyCore
  `tc@box:~$` at 3000 frames, native == sandbox on state and video. The disc
  path was gated byte-exact to 6000 frames; the shell *via disc* was later
  booted to completion once the IDE pacing landed (see below).
- **M3 - a real core package, booted through the frontend - DONE (0048c24).**
  `qemu.chimeraCore` (core.wbx + waterbox.config + keybinds + file_slots
  {disc, kernel, initrd} + SeaBIOS assets + licences + build.json provenance),
  `PC` in SystemNames. Verified at every layer, each byte-exact against
  run-wbx's machine: engine `chimera-run` (disc leg: 256 MiB pc.ram at 1200
  frames; kernel leg: 1 MiB pc.ram at 3000 frames) and the real frontend
  (mono Chimera.exe headless, `run-frontend.sh`: 3100 frames with a held key,
  `frontend-dump.lua` driving, pc.ram identical). Interactive typing in the
  GUI confirmed by the user.

## Performance: the disc problem - SOLVED (patch 0010)

Disc boot *was* firmware-polling-bound: the block filter parked every
request in a coroutine that only the next frame-boundary vm_stop released,
so each disc command cost a full frame of firmware polling, and the whole
disc phase crawled (15-25 min to a shell). The structural fix, IDE-level
pacing, is now in:

- The filter (`chimera-latency.c`) is COW + streaming only; it no longer
  imposes latency of any kind.
- Every async IDE block submission (`ide_buffered_readv`, the disk PIO/DMA
  paths, flush) is **drained to its host completion at the submission
  instant** and the completion **parks** in a per-bus queue. A
  `QEMU_CLOCK_VIRTUAL` timer delivers it at **deadline = (the guest's last
  IDE port access, or the previous delivery) + `CHIMERA_IDE_LATENCY_NS`, set to 0** -
  with the drain guaranteeing park-before-deadline, determinism holds at any
  latency; 0 avoids a ~100k-instruction poll spin per request at shift=0.
  All three anchors are pure virtual instants - a port access runs on the
  vCPU under icount, the drain and the timer callback run with the vCPU
  quiescent (icount forces the single-threaded vCPU) - so both builds
  deliver at the same virtual nanosecond. A firmware poll loop now spins
  microseconds of virtual time instead of a whole frame boundary; a
  halted guest reaches the deadline through the warp governor like any
  other virtual timer.
- The drain-at-submission is the load-bearing detail, learned the hard way:
  parking at host completion time let completions *slip* past their
  deadline (52 slips per disc boot on the reference host at the original 100µs
  latency - warp steps jumped past deadlines while I/O was in flight), which made delivery
  land at host-chosen instants and broke native-vs-native determinism; and
  the sandbox's cooperative threads cannot service a mid-frame aio
  completion at all (the guest polls `Status 0xd0` forever - the
  green-thread main loop only runs block completions at the frame
  boundary). Draining at submission fixes both: the park is always before
  the deadline, and no mid-frame aio service is ever needed. The
  synchronous ATAPI PIO path (`blk_pread` - the SeaBIOS/El Torito reads)
  always worked this way, which is why firmware disc reads never needed
  pacing.
- Resets, cancels and the save path force-deliver first (`chimera_ide_flush_pending`
  is called by the driver before every snapshot), so a parked completion -
  a function pointer - can never enter the migration stream. The pacing
  state lives on `IDEBus` and is deliberately outside any vmstate.

Result, measured on the reference machine: disc boot (SeaBIOS → El Torito
→ ISOLINUX loads 26 MB → kernel → live shell answering keys) is **~1m50s
wall**, native A == B == sandbox byte-exact, savestates round-trip. The
kernel leg's gate got faster too (~30s wall for 3000 frames).

Parked oddity RESOLVED for TinyCore: the ISOLINUX `boot:` timeout does fire
under icount (30 virtual seconds = frame ~1800), so the disc boots
unattended; the earlier Alpine observation (no timeout in 100 virtual
seconds) remains unexplained but Alpine-specific - revisit if Alpine
becomes the full-distro vehicle.

## After that: a full distro and software

With disc boot in ~2 minutes, the natural ladder is Alpine (or a small
Debian) with the `kernel`+`initrd`+`disc` slots all filled - the initrd
boots, the disc mounts as the root filesystem's source - then run something
real in it (a shell script, a compiler, a game) under movie control. The COW
overlay already persists across savestates, so "install once, boot the
installed system" is expressible as a movie even before persistent disk
export exists.

## Deferred

Audio (AC97/SB16 need the xemu-APU virtual-clock treatment), mouse/tablet,
persistent disk export (the COW overlay is not yet a file), q35/`-machine`
setting, i386 distros.

## Handoff: how to run everything

- **GUI:** `~/chimera/build/ChimeraMono.sh`, Open Project →
  `~/chimera/projects/tinycore-direct/tinycore.chimeraProject`. Boot is ~1
  minute; type at the PC's keyboard. For disc boot, copy that project and
  swap the files to one entry naming the ISO with `"slot": "disc"` - it
  boots to the shell in ~2 minutes (ISOLINUX waits 30 virtual seconds at
  its `boot:` prompt unless Enter is pressed).
- **Package:** `sh ~/chimera-core-qemu/waterbox/build-package.sh` →
  `~/chimera/build/Cores/qemu.chimeraCore` (deterministic zip; sha1 printed).
- **The gate:** `sh ~/chimera-core-qemu/waterbox/run-gate.sh [frames]` (A==B==
  sandbox); `CHIMERA_KERNEL=… CHIMERA_INITRD=…` env knobs for the kernel leg
  on native, `--kernel/--initrd` on run-wbx.
- **The frontend check:** `sh ~/chimera-core-qemu/waterbox/run-frontend.sh`
  (mono headless vs run-wbx, pc.ram byte-compare; `--work DIR` keeps dumps).
- **Engine harness:** `~/chimera/build/meson-linux/chimera-run --project <p>
  <package> [--frames N] [--dump pc.ram=F] [--screenshot F=F.tga]`. Movie
  entries are `|` + one char per declared button (161 for this core) + `|`.
- Native debugging knobs: `CHIMERA_PREVIEW=1` (no icount, direct disc,
  free-running, NOT deterministic - eyes only; needs a display for SDL),
  `CHIMERA_SDL=1`, `CHIMERA_TRACE_EXEC=<path>` (+ `CHIMERA_TRACE_SPEC` for
  IDE events).
- Boot files for the default project: extract `boot/vmlinuz64` and
  `boot/corepure64.gz` from the TinyCore ISO (`build/discs/corepure64-17.1.iso`
  in the core repo; bsdtar) - they are machine inputs, so treat them as
  pinned.
