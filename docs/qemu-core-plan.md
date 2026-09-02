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
checked. What remains is performance (the disc path) and, beyond it, a full
distro running real software.

- Core repository `chimera-core-qemu` (mirrors `chimera-core-xemu`), submodule
  at `extern/cores/qemu`, nested `extern/qemu` pinned to the same QEMU release
  family xemu rides (10.2.x).
- The frontend's share is one line (`PC` in SystemNames); everything else
  rides the package, like xemu.

## Machine identity (decided, movies cite it)

| Knob | Value |
|---|---|
| Machine / CPU | `pc` (i440fx), 1 vCPU, TCG, `-cpu qemu64` (no `rdrand`/`rdseed`) |
| icount | `shift=2` (a 250MHz-class virtual CPU) - halves firmware poll cost per frame |
| Governor | period 1ms (`GOVERNOR_NS 1000000`) |
| Boot order | `-boot order=d` (don't walk floppy/hard disk first) |
| Frame quantum | VGA 60 Hz: `VBLANK_NS 16666667`, vsync `1000000000/16666667` |
| RTC | `base=2000-01-01,clock=vm` |
| RAM | 256 MiB default (`ramMiB` setting: 128/256/512) |
| Memory layout | `[512, 16, 128, 256, 2048]` MiB (xemu's numbers, proven at this scale) |
| Video | VGA scanout per frame → pixman → BGRA; text mode included |
| Input | PS/2 keyboard, buttons in QKeyCode order, press/release diffs at the paused frame boundary |
| Firmware | SeaBIOS + VGABIOS + linuxboot_dma ride the package **assets channel** (LGPL, shippable) |

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
   disc traffic at all. TinyCore reaches its shell in 3000 frames (~1 min
   wall). This is the default project (`~/chimera/projects/tinycore-direct`)
   and the iteration path.
2. **Disc boot (El Torito through SeaBIOS).** The project fills the `disc`
   slot. Deterministic and gated, but slow - see the performance section; this
   is the blocking work item.

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
  path is gated byte-exact to 6000 frames (deep into Linux userspace); the
  shell *via disc* was never waited out - that is the performance item below.
- **M3 - a real core package, booted through the frontend - DONE (0048c24).**
  `qemu.chimeraCore` (core.wbx + waterbox.config + keybinds + file_slots
  {disc, kernel, initrd} + SeaBIOS assets + licences + build.json provenance),
  `PC` in SystemNames. Verified at every layer, each byte-exact against
  run-wbx's machine: engine `chimera-run` (disc leg: 256 MiB pc.ram at 1200
  frames; kernel leg: 1 MiB pc.ram at 3000 frames) and the real frontend
  (mono Chimera.exe headless, `run-frontend.sh`: 3100 frames with a held key,
  `frontend-dump.lua` driving, pc.ram identical). Interactive typing in the
  GUI confirmed by the user.

## Performance: the disc problem (the next work item)

Disc boot is firmware-polling-bound. SeaBIOS (and ISOLINUX) poll the IDE
status port while every request sits at the frame-boundary latency the
determinism filter imposes; each poll iteration is emulated device traffic,
so the wall-clock cost is (poll iterations) x (icount cost), and the whole
disc phase crawls (~15-25 min to a shell; kernel compute frames are ~8x
cheaper than firmware poll frames). xemu never paid this tax: the Xbox kernel
halts on IRQ instead of polling. Raising icount shift halves poll cost but
stretches virtual boot time; the governor/shift tuning already done took a
frame from ~2.6s to ~0.63s - further knob-turning has hit diminishing returns.

The structural fix, deferred until now by design: **IDE-level pacing** -
anchor disc completions to a short *fixed virtual delay* measured at the
device-instruction instant, instead of the block filter's frame-boundary
quantisation. It must live at the IDE device (the block layer's clock reads
are host-fuzzy and nondeterministic); the block filter stays for the COW
overlay and streaming, but stops being the latency source. Design sketch:
intercept the IDE command write in `hw/ide/core.c`, take `qemu_clock_get_ns(
QEMU_CLOCK_VIRTUAL)` then, schedule the completion `N` virtual ns out; poll
loops then spin over a bounded virtual window (~microseconds of virtual time)
instead of a whole frame boundary. The gate re-verifies everything (A==B==
sandbox, disc legs re-run byte-exact - machine identity changes, which is
fine pre-1.0).

Parked oddity from the disc experiments, for the record: under icount the
disc path sat at Alpine's ISOLINUX `boot:` prompt for 100 virtual seconds
where full-speed runs time out in ~20s - unexplained, worth revisiting when
the IDE pacing work touches that path.

## After that: a full distro and software

Once disc boot (or a disc-free equivalent: an installed-to-COW hard disk, or
a bigger initrd) boots in minutes, the natural ladder is Alpine (or a small
Debian) with the `kernel`+`initrd`+`disc` slots all filled - the initrd boots,
the disc mounts as the root filesystem's source - then run something real in
it (a shell script, a compiler, a game) under movie control. The COW overlay
already persists across savestates, so "install once, boot the installed
system" is expressible as a movie even before persistent disk export exists.

## Deferred

Audio (AC97/SB16 need the xemu-APU virtual-clock treatment), mouse/tablet,
persistent disk export (the COW overlay is not yet a file), q35/`-machine`
setting, i386 distros.

## Handoff: how to run everything

- **GUI:** `~/chimera/build/ChimeraMono.sh`, Open Project →
  `~/chimera/projects/tinycore-direct/tinycore.chimeraProject`. Boot is ~1
  minute; type at the PC's keyboard.
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
