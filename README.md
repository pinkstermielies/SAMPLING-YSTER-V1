# PINKSTERKEYZ SAMPLE MACHINE

Native Windows SoundFont (SF2) sampling workstation. C++20 · JUCE 8 · CMake · GitHub Codespaces · GitHub Actions.

> **Status: Stage 1 of 7.** This repository is being built in verified stages. The table below says exactly what
> works today and how it was checked. Nothing is mocked; anything not built yet says so in the UI.

## What exists today

| Module | Status | How it was verified |
|---|---|---|
| **SF2 writer** (`Source/SoundFont`) - RIFF `sfbk`, INFO/sdta/pdta, phdr/pbag/pmod/pgen/inst/ibag/imod/igen/shdr, 46-point padding, loop points, global zones, key/vel ranges, generator ordering, atomic file write | Complete | Unit tests + built-in round-trip verifier + an **independent Python parser** (`scripts/verify_sf2.py`). Not yet loaded in a real player (see below). |
| **Key mapping** model (overlaps allowed, lock, range edit) | Complete (model) | Unit tests |
| **Velocity layers** (even split, overlap prevention, gap/overlap validation, boundary dragging) | Complete (model) | Unit tests |
| **Auto-Fill Open Notes** (Nearest / Balanced / Manual Fine-Tune, preview, per-layer, never touches existing or locked zones) | Complete (engine) | Unit tests, ASan + UBSan |
| **Loop maths** (validation, zero-crossing snap, equal-power crossfade, Ping-Pong unroll, Reverse render, One-Shot, SF2 sampleModes) | Complete (engine) | Unit tests incl. "export sounds identical to editor preview" |
| **Sampling plan** (interval, velocity layers, round-robin, naming) | Complete | Unit tests |
| Undo/redo (`History<T>`) | Complete | Unit test |
| **Audio engine** - device manager, peak meters, monitoring, lock-free recording to 24-bit WAV, threshold start, max length | Written, **not yet compiled** | Needs the Windows CI build |
| **MIDI router** - in/out selection, note on/off, program/bank, all-notes-off | Written, **not yet compiled** | Needs the Windows CI build |
| UI shell - dark/gold theme, 13-section navigation, HOME (meter + recorder), AUDIO SETTINGS, MIDI SETTINGS | Written, **not yet compiled** | Needs the Windows CI build |

**Not built yet:** auto-sampler run loop, VST3 host, sample manager / project format, waveform editor, graphical
velocity / key-range / loop editors, sound-design knobs, SF2 export page, WASAPI loopback capture.
Their pages show "Not implemented yet" rather than pretending.

## Build

### Anywhere (no JUCE, no network): portable core + unit tests
```bash
./scripts/test-core.sh          # plain g++
# or, with CMake:
cmake --preset linux-core && cmake --build --preset linux-core && ctest --preset linux-core
```

### GitHub Codespaces
Open the repo in a Codespace. The container installs the toolchain and JUCE's Linux dependencies and runs
`scripts/build-linux.sh` automatically. For a Linux *development* build of the app: `./scripts/build-linux-app.sh`.
Codespaces is Linux and cannot produce the Windows EXE - that is done by GitHub Actions.

### Windows EXE (GitHub Actions)
Push to `main` (or run **windows-build** manually). The workflow builds Release x64 with MSVC, runs the tests,
and uploads `PinksterKeyzSampleMachine.exe`, a zip, and an Inno Setup installer as artifacts.

### Windows locally (Visual Studio 2022)
```powershell
cmake --preset windows-release
cmake --build --preset windows-release
ctest --preset windows-release
```

## Audio drivers
* **WASAPI and DirectSound**: enabled.
* **ASIO**: off by default - Steinberg's ASIO SDK cannot be redistributed. Download it yourself, then configure with
  `-DPSM_ENABLE_ASIO=ON -DPSM_ASIO_SDK_DIR=C:/path/to/ASIOSDK`. (Many interfaces also work well via WASAPI.)
* **Software-instrument capture from other programs**: the planned route is a virtual audio cable (e.g. VB-CABLE) selected as an
  input; a dedicated WASAPI loopback capture is planned for stage 3.

## Roadmap
1. **Core engine** - this stage.
2. Auto-sampler run loop (MIDI out -> record -> trim -> name), pause/resume/cancel, progress UI.
3. VST3 host + plugin sampling, loopback capture.
4. Project format (save/open/recent/autosave/recovery) + sample manager + waveform editor.
5. Velocity-layer, key-range and Auto-Fill editors (graphical 128-key keyboard).
6. Loop editor + sound-design (ADSR, filter, tuning) UI.
7. SF2 export page (whole project / selected instrument / WAV export, metadata), player compatibility testing, installer polish.

## Layout
`Source/<Module>/` mirrors the spec: Audio, MIDI, Recording, PluginHost, Sampler, Waveform, LoopEngine, Velocity,
KeyMapping, AutoFill, SoundFont, Project, UI. Modules that need no JUCE are compiled into `psm_core` so they are
unit-tested on every platform.

## Real-time rules
`AudioEngine::audioDeviceIOCallbackWithContext` and `RecordingEngine::process` never allocate, never touch files, and
never block (try-lock + lock-free FIFO to a background writer thread; a skipped block is counted, not waited for).

## Licensing
`LICENSE` is a placeholder. JUCE is AGPLv3 or commercial/personal-licensed; pick the option that fits how you intend to
distribute this application before publishing.
