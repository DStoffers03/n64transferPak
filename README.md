# n64transferPak

Status: implementation plan, 28 October 2026. The preparation tool and its tests are not implemented yet. Emulator integration and game compatibility have not been tested for this project.

The initial repository baseline is commit `23c9eb1e51914047c3514facf6fc3aa17b104c40`, whose tree contains only `LICENSE`. This README defines the first implementation slice toward the repository's stated direction: “Adding transfer pak support for mupen64.”

## Goal and first milestone

Support a manual iPhone-to-Mac workflow: save in a GB/GBC game on iOS, export its battery save, transfer it with AirDrop, and let a macOS N64 emulator access a copy through an emulated Transfer Pak.

The proposed first implementation is one offline command that prepares a disposable, byte-for-byte ROM/save session for controller 1. It handles files and their provenance; existing emulator code handles the virtual cartridge.

The first candidate game pair is Pokémon Crystal with Pokémon Stadium 2. This is a proposed test case; the actual ROM regions/revisions and compatibility remain unverified. A supported result requires testing the actual pair.

Success for this milestone means the staged files match the originals and, separately, a recorded Mac smoke test shows Stadium recognizing the expected saved trainer/party. Copy success alone is not emulator compatibility.

## Emulator baseline and security choices

| Role | Proposed baseline | Evidence and qualification |
| --- | --- | --- |
| iOS GB/GBC | Delta, installed through its official App Store listing | Delta's codebase is AGPLv3; its GBC core is Gambatte. The official guide links the App Store distribution. No jailbreak or sideloading is needed for this route. |
| macOS N64 | Stable RetroArch with Mupen64Plus-Next | RetroArch is GPLv3. Current Mac installation documentation provides a Universal build for Intel and Apple Silicon. Mupen64Plus-Next documents Transfer Pak ROM/save loading. Exact installed core, rendering configuration, and game behavior remain to be tested. |
| File transfer | Manual AirDrop to your Mac | No custom server, cloud account, background sync agent, or network protocol is part of this project. |

This is the recommended security-conscious baseline, not an audited ranking of every emulator. Open source permits inspection; it does not by itself establish security. Use official distributions, keep OS/app security updates current, and retain Gatekeeper checks. Libretro has described its Mac releases as signed and notarized; the actual downloaded build still needs normal macOS verification. Do not disable Gatekeeper or remove quarantine to get an unexpected build running.

Emulator choice remains replaceable. As of 28 October 2026, OpenEmu's Mupen64Plus-Core Transfer Pak request remains open. That issue is a reason to verify the frontend boundary, not proof that every current OpenEmu build is incapable. The documented RetroArch path is the first baseline to test.

## Smallest useful repository

| Path | Purpose |
| --- | --- |
| `LICENSE` | Existing GPLv3 license; preserve it. |
| `README.md` | This scope, setup, contract, evidence, and verification record. |
| `.gitignore` | Exclude local sessions, ROMs, saves, clocks, and Python artifacts. |
| `transferpak.py` | One Python 3.12+ standard-library CLI for preparing a session. |
| `tests/test_transferpak.py` | File-integrity and failure-path tests using temporary synthetic bytes. |

This README records the plan. Next, add the ignore rules and the two Python files for the implementation slice. No dependency manifest is necessary while the program has no third-party packages.

Proposed `.gitignore`:

```gitignore
/local/
__pycache__/
*.py[cod]
.venv/
.DS_Store
*.gb
*.gbc
*.z64
*.n64
*.v64
*.sav
*.srm
*.rtc
*.state*
*.svs
```

Keep real ROMs and saves outside the checkout; ignore rules are an additional safeguard. Tests generate synthetic data at runtime and commit no game binaries.

## First interface: file images into the emulator's media loader

Model this boundary:

`prepare_session(n64_rom, gb_rom, gb_save, out_dir) -> SessionManifest`

The arguments are explicitly selected local paths. The result names a newly created session directory and its integrity manifest.

This is the handoff from files to the existing emulated cartridge, immediately before the emulator opens the GB ROM and battery RAM. It is not an N64 controller-bus API or a new mapper implementation.

The inspected Mupen64Plus-Next source already supplies `retro_transferpak_rom_path` and `retro_transferpak_ram_path` to ROM/RAM initialization; media-loader callbacks include `get_gb_cart_rom` and `get_gb_cart_ram`. Treat these as implementation references, not a stable public API to link against in this slice.

Contract:

1. Accept one uncompressed `.z64` N64 image, one `.gb` or `.gbc` image, and one explicitly selected `.sav` battery-save export. Limit the adapter to controller 1.
2. Read source files without modification. Require nonempty regular files; reject symlinks, directories, missing inputs, recognized archive/save-state extensions, and an existing output path.
3. Exclusively create the destination directory and create ordinary independent copies, never hard links or symbolic links.
4. Stage the inputs as `game.z64`, `game.z64.gb`, and `game.z64.sav`. The name change is an emulator loading convention; it does not convert file contents.
5. Stream the copies, record byte lengths and SHA-256 hashes, verify destination bytes, and detect source changes during copying. Write the success manifest last. A failure must not leave a success manifest or alter an input.
6. Return status 0 only for a completed, integrity-checked bundle; use status 2 for invalid inputs or an existing destination, and status 3 for I/O or integrity failures. Print a concise error to stderr.
7. Make no network requests, execute no input, launch no emulator, and modify no emulator configuration. Leave any actual emulator writes confined to the disposable copies.

The manifest needs only `schema_version: 1`, `controller: 1`, `adapter: "retroarch-mupen64plus-next"`, `compatibility: "unverified"`, `rtc: "unverified"`, and a file entry for each role containing source basename, staged filename, byte length, and SHA-256. Avoid absolute personal paths in the manifest.

The helper verifies copying, not semantic save validity. A filename ending in `.sav` cannot establish game identity, save format, or compatibility. It must not silently pad, truncate, repair checksums, swap byte order, or convert an unknown save. Hash equality establishes byte identity, not trustworthiness of the game image.

“Read-only” here applies to source files. Existing emulator code may perform mapper/register writes and may save battery RAM into the staged copy; suppressing all virtual writes could break normal behavior.

## Setup and manual baseline before further emulator coding

### Record the environment

Fill in the verification table below rather than guessing versions. Capture macOS version and CPU architecture with:

```sh
sw_vers
uname -m
```

Install Delta through the App Store link in its official guide. Install the stable Universal RetroArch build through its official macOS instructions, then obtain Mupen64Plus-Next through RetroArch's official core distribution. Record frontend version, core version/build identifier, and the loaded core binary's SHA-256.

The current Mac guide recommends `glcore` or Vulkan rather than the experimental Metal video driver. Start with `glcore` for this baseline, record the selected RDP/RSP and CPU settings, and verify that the N64 game boots. Availability of the frontend alone does not prove that the exact N64 core/configuration runs correctly on this Mac.

### Export an actual battery save

1. Save using the GB/GBC game's own save command, then return to Delta's library.
2. Long-press the game and use **Manage Save File → Export Save File**.
3. AirDrop the export to your Mac. Retain a dated original backup and keep it outside the emulator session.
4. Use the exact GB/GBC ROM revision that produced that save. Record region, language, revision, byte length, and hash for the ROM and save.
5. Preserve any available RTC sidecar separately. The first helper neither imports nor converts RTC data.

Delta's GBC bridge explicitly reads and writes a separate `.rtc` file beside its save. That source observation does not establish whether the currently installed Delta share/export flow includes it. Verify the actual export. Do not substitute a save state for an in-game battery save.

### Prove the existing Mac path

Use three disposable copies in one directory, named `game.z64`, `game.z64.gb`, and `game.z64.sav`. These follow the documented Mupen64Plus-Next filename convention.

Load the N64 game, set **Quick Menu → Options → Player 1 Pak → transfer**, then close content and reload `game.z64` with Mupen64Plus-Next. Menu placement can vary by frontend/core version; record the actual route. If the core or option is absent, mark this baseline blocked rather than treating that build as supported.

In Stadium, check the expected trainer/party using the copied cartridge data. Do not use a round-trip write test for this milestone. Exit the emulator normally and verify that the retained original hashes are unchanged. The session save may have changed and is disposable.

### Developer setup for the proposed helper

Prerequisite: a maintained Python 3.12+ installation, available as `python3`; use the official Python macOS distribution if needed. No pip packages or elevated privileges are required.

The following project commands become usable only after the proposed Python files are implemented:

```sh
git clone https://github.com/DStoffers03/n64transferPak.git
cd n64transferPak
python3 --version
python3 -m unittest discover -s tests -v
python3 transferpak.py prepare \
  --n64-rom "/absolute/private-path/Stadium2.z64" \
  --gb-rom "/absolute/private-path/Crystal.gbc" \
  --gb-save "/absolute/private-path/Crystal.sav" \
  --out-dir "/absolute/private-path/session-001"
```

Replace example input paths with actual files. The output directory must not exist. These commands and filenames are a proposed interface, not software already present in the repository.

## Required checks and completion criteria

| Check | Required result |
| --- | --- |
| Successful preparation | Three distinct copies, matching input lengths/hashes, manifest written last. |
| Protection of originals | Identical source bytes before/after; no links to originals. |
| Failure handling | Missing/empty/nonregular input, existing destination, source change, and copy failure cannot produce success. |
| Test isolation | Synthetic temporary inputs; no ROM downloads, network, emulator installation, or copyrighted fixtures. |
| macOS smoke test | Recorded Delta/RetroArch/core/OS/architecture combination recognizes the expected trainer/party from a disposable session. |
| Scope of success | Read/import path only. No claim of clock preservation, full GB Tower operation, transfers back to iOS, or hardware fidelity. |

File tests establish utility correctness. They cannot establish Transfer Pak emulation accuracy. The Mac smoke test has not been run for this proposal.

## Verification record and unresolved boundaries

| Item | Current status | Evidence needed |
| --- | --- | --- |
| Initial repository baseline | Verified 28 October 2026 | Initial commit above; its tree contains LICENSE only. |
| Delta iOS build and device OS | UNVERIFIED | Installed versions and exact game-save export procedure. |
| Mac OS/CPU/frontend/core build | UNVERIFIED | Versions, architecture, core origin/hash, settings, and boot result. |
| Crystal/Stadium 2 region/revision pair | PROPOSED; UNVERIFIED | User-supplied exact images and an observed compatibility test. |
| Battery-save layout and size | UNVERIFIED | Actual Delta export plus selected core's source/behavior; no universal “correct .sav size” assumed. |
| RTC export/import and clock behavior | UNVERIFIED | Whether export includes sidecar, format compatibility, persistence and round-trip tests before enabling writeback. |
| Save persistence in chosen Mac build | UNVERIFIED | Determine which file the selected core writes and when; inspect only disposable copies. |
| Hardware protocol accuracy | OUT OF SCOPE | Before bus-level changes: independently sourced/observed register semantics, banking, initialization/reset, transaction size, timing, CRC, and mapper behavior. Existing emulator code is not proof of physical behavior. |
| Physical adapters/controllers | NO REQUIREMENT ESTABLISHED | A real-device model, electrical interface and test setup would be needed before any USB/GPIO/physical Transfer Pak work. |

Do not implement custom hardware commands, hard-force a controller type in OpenEmu, or hard-code a personal ROM/save path from this proposal. The setup exposes a virtual Transfer Pak through existing software. Keep save writeback, RTC conversion, live cross-device connections, and additional controllers for separately verified later work.

## Source anchors

Checked 28 October 2026. Source inspection is evidence of implementation, not a completed device/game test.

- [Repository](https://github.com/DStoffers03/n64transferPak) and [exact initial tree](https://github.com/DStoffers03/n64transferPak/tree/23c9eb1e51914047c3514facf6fc3aa17b104c40).
- [Delta source, systems and AGPLv3 license statement](https://github.com/rileytestut/Delta).
- [Delta official distribution links](https://faq.deltaemulator.com/).
- [Delta official save export/import instructions](https://faq.deltaemulator.com/using-delta/nintendo-ds/incompatible-save-states). This page documents the menu in a DS migration context; the exact GBC export remains a test item.
- [Delta GBC save/RTC bridge at inspected commit](https://github.com/rileytestut/GBCDeltaCore/blob/0871ccaad2bbd7cbd2de0ce06f9e26dc3d1bfdde/GBCDeltaCore/Bridge/GBCEmulatorBridge.mm#L269).
- [RetroArch source and GPLv3 license](https://github.com/libretro/RetroArch).
- [Current RetroArch macOS installation instructions](https://docs.libretro.com/guides/install-macos/).
- [Libretro's Mac signing/notarization announcement](https://www.libretro.com/index.php/retroarch-for-apple-mac-m1-softlaunches-today/). Historical distribution statement; verify the actual current artifact.
- [Mupen64Plus-Next Transfer Pak setup](https://docs.libretro.com/library/mupen64plus/#using-the-transfer-pak).
- [Mupen64Plus-Next file selection at inspected commit](https://github.com/libretro/mupen64plus-libretro-nx/blob/15bada85906593376e8fd697f1b0b3af3a20f512/libretro/libretro.c#L1907).
- [Mupen64Plus-Next GB ROM/RAM initialization at inspected commit](https://github.com/libretro/mupen64plus-libretro-nx/blob/15bada85906593376e8fd697f1b0b3af3a20f512/mupen64plus-core/src/main/main.c#L1231).
- [OpenEmu Mupen64Plus-Core Transfer Pak request #35](https://github.com/OpenEmu/Mupen64Plus-Core/issues/35).
- [Python macOS downloads](https://www.python.org/downloads/macos/).

Preserve the repository's existing LICENSE. This proposal vendors no emulator code and includes no ROMs or user saves.
