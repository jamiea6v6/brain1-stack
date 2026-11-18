# brain1-stack — the local AI agent stack, captured so it can be rebuilt

A $30 mining card, a 4B model, and a harness that makes the model check its own work.

This repo holds the **configuration half** of the stack that runs on `brain1` (192.0.2.86): the Pi
coding agent's harness, the model/provider config, the Ollama `/v1` shim, and the manifest that
explains every version pin. It exists so the stack can be **rebuilt from scratch on other hardware**
— Trevor's box, a second VM, a spare machine — without rediscovering why each piece is the way it is.

The **grader half** (a deterministic test harness, a fake sound card, six deliberately broken mutants)
lives in the `cw-dsp` repo on brain1. That split is deliberate: the stack is general, the grader is a
worked example of how you grade a model that cannot be trusted to grade itself.

---

## What is in here

```
MANIFEST.md            every version pin + why it is pinned  ← read this first
harness/extensions/    the 9 Pi extensions (the actual IP)
harness/models.json    provider, model, context window, sampling
harness/pi-AGENTS.md   the harness rules (short on purpose — every line costs per turn)
harness/settings.json  pi's own settings
systemd/               the shim's user unit
tools/                 the Ollama /v1 shim (why it exists: /v1 ignores num_ctx)
tests/                 the pre-registered local-agent test suite (snapshot)
```

## REPOS REQUIRED FOR A NEW INSTALL

**Three.** All three are bare repos under `~/repos/` on SkyNet (192.0.2.204). Each
clone checks out its own branch — `brain1-stack` is `main`, the other two are `master`.

| Repo | What it is | Clone to |
|---|---|---|
| `brain1-stack` | the harness + model config + shim + MANIFEST (**this repo**) | `~/projects/brain1-stack` |
| `mcu-domain` | DOMAIN.md, the two gates, the stage template, `refs/`, `projects/` | `~/projects/mcu-domain` |
| `cw-dsp` | the grader half: `spec_cw_filter.py`, `stub_pyaudio.py`, `mutants/`, FRIEND-KIT.md | `~/projects/cw-dsp` |

```sh
git clone ~/repos/brain1-stack.git ~/projects/brain1-stack
git clone ~/repos/mcu-domain.git  ~/projects/mcu-domain
git clone ~/repos/cw-dsp.git      ~/projects/cw-dsp
```

Not a repo but required on the box: **`arduino-cli` 1.6.0** user-local (`~/bin/arduino-cli`,
installed on brain1 and SkyNet) plus a board core — `arduino-cli core install arduino:avr` for the
Uno, and the third-party ESP8266 core for a Wemos D1.

**Not needed, and must not be copied:** the homelab doc store. brain1's `~/AGENTS.md` symlinks to
The author's personal context — 2340 tokens on every turn, and none of it is about the stack. The kit
ships a slim `harness/pi-AGENTS.md` instead.

## The three ideas this encodes

1. **The model must never be the judge of its own output.** A separate process decides pass/fail;
   the model reads the failures and fixes the case named. That is what `verify` is for, and what
   `cw-dsp`'s `spec_cw_filter.py` is an example of.
2. **Measure the harness, not the vibe.** The test suite fixes prompts, ground truth and PASS/FAIL
   *before* the run, so a result is falsifiable instead of a matter of opinion.
3. **Context is the scarce resource, not VRAM.** 32K tokens of context costs 228 MiB of VRAM on a
   6 GB card. The prompt budget is what runs out.

## Deploying

```bash
git clone <this repo> brain1-stack
cd brain1-stack
./bootstrap.sh          # coming — Wednesday
./verify-stack.sh       # coming — Wednesday; prints PASS/FAIL per check
```

Until `bootstrap.sh` lands, the manual sequence is in `MANIFEST.md` §2–3. The short version:
NVIDIA driver **550.163.01** → Ollama **0.32.15** → Node **22** in `~/.local/node` → Pi **1.1.0** →
copy `harness/` into `~/.pi/agent/` → install the shim unit → pull `gemma4:e4b-it-qat` →
`apt install libportaudio2`.

## Two known bugs in the captured harness

Fixed in the source of truth, not yet in this capture — see `MANIFEST.md` §6:

- `context-budget.ts` still assumes an 8192-token window while the shim provides 32768, so it trims
  ~2340 tokens per turn for a window the box does not have.
- `verify-package.ts` reports "DOES NOT EXIST" when its probe fails, not only when a package is
  genuinely absent (it produced three false negatives on numpy/scipy/sounddevice).

## Provenance

Captured 2026-11-14 from the live brain1 stack. All 14 captured files were sha256-verified
byte-identical to their live counterparts — the table is in `MANIFEST.md` §4.
