# Fibre-Coupling Alignment Automation with MCP

A prototype software framework for **automating fibre-coupling alignment**, built
on the **Model Context Protocol (MCP)**.

Fibre coupling in the optical setup is normally aligned by hand, which is slow and
hard to reproduce. This project drives the alignment — the tilt mirrors, a
coupling-efficiency model, and a power meter — through a single programmatic
interface, so it can be run and repeated in software instead of manually.

---

## What this prototype does

The fibre-alignment loop targets fibre-coupling efficiency through three pieces,
coordinated by an orchestrator:

1. **Optics** — Thorlabs XA tilt mirrors (2 angular DOF each, θ/φ in millidegrees).
2. **Surrogate "digital twin"** — a small neural network that predicts coupling
   efficiency η from the current optical state, so the optimizer can propose mirror
   moves cheaply without hammering the hardware.
3. **Power meter** — a Thorlabs power meter that provides the **ground-truth**
   coupling measurement, used to validate the twin's predictions against reality.

The orchestrator runs a closed-loop step: read the live mirror state → optimize
mirror angles in the surrogate's input space → command the mirrors → (optionally)
measure the real power before/after to check the twin against the bench.

---

## Status at a glance

| Component | File | Status |
|---|---|---|
| Tilt-mirror control (Thorlabs XA, pythonnet/.NET) | `mcp_servers/optics_server.py` | **Implemented** (hardware + simulation) |
| Power meter (Thorlabs PM, PyVISA/SCPI) | `mcp_servers/power_meter_server.py` | **Implemented** (hardware + simulation) |
| Fibre-coupling surrogate NN (digital twin) | `mcp_servers/surrogate_fiber_coupling.py` | **Implemented** (trained model included) |
| Orchestration / closed-loop alignment | `mcp_servers/orchestrator_server.py` | **Implemented** |
| CLI helpers for optics | `utils/` | **Implemented** |
| Stage repeatability study + logged data | `tests/thorlabs_xa_repeatability_study.py`, `repeatability_logs/` | **Implemented** |
| Laser (Cobolt), spectrometer, camera, safety | `mcp_servers/{laser,spectrometer,camera,safety}_server.py` | **Planned** — empty stubs reserved for next terms |

Being explicit so the scope is clear: this is a **prototype**. The fibre-alignment
components and the closed-loop routine are implemented in code, but a full
end-to-end run has **not** been demonstrated and the loop is not
validated against hardware. The laser / spectrometer / camera / safety servers are
empty placeholders, out of scope for this project and **not implemented**.

---

## Repository map

```
mcp_servers/
  optics_server.py              # Thorlabs XA tilt mirrors; θ/φ in millidegrees; .NET via pythonnet
  power_meter_server.py         # Thorlabs power meter over PyVISA (SCPI); ground-truth coupling
  surrogate_fiber_coupling.py   # FiberCouplingNet (9→64→1024→1); predicts coupling efficiency η
  orchestrator_server.py        # Combines optics + surrogate + power meter; closed-loop alignment
  dotnet_cast.py                # pythonnet .NET interop helper
  {laser,spectrometer,camera,safety}_server.py   # planned (empty stubs)

surrogate-NN/
  Update of Automated optical alignment using digital twins.ipynb   # trains/explains the twin
  NN_parameter_data/            # trained weights (fiber_coupling_model.pth) + normalisation params
  sample_data/                  # example input maps for the twin

utils/                          # CLI wrappers: set_mirror_angle, sweep_mirror, move_steps (see utils/README.md)
tests/                          # unit + integration tests, plus the XA repeatability study
repeatability_logs/             # logged results of the repeatability study (CSV + provenance)
design_choices.md               # dev-environment decisions (uv, local-vs-Colab, VISA notes)
Miscellaneous/                  # spring notes, scratch notebooks, standalone experiments
```

The package name is `mcp-diamond-spectroscopy` (differs from the repo folder name).

---

## Running it (no bench hardware required)

The implemented components are meant to run on a normal machine in **simulation
mode** — the optics and power-meter servers fall back to in-process stubs, so they
can be exercised without the Thorlabs hardware or a VISA runtime.

**Setup** (uses [`uv`](https://github.com/astral-sh/uv); Python ≥ 3.12):

```bash
git clone https://github.com/jaranmio/Autonomous-Fibre-Alignment-with-MCP.git
cd Autonomous-Fibre-Alignment-with-MCP
uv venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
uv sync
```

**Run the test suite in simulation:**

```bash
OPTICS_SIMULATE=1 PM_SIMULATE=1 uv run pytest -q
```

**Drive the optics directly** from the CLI (simulation):

```bash
OPTICS_SIMULATE=1 uv run utils/set_mirror_angle.py --theta-mdeg 100 --phi-mdeg 0
OPTICS_SIMULATE=1 uv run utils/sweep_mirror.py --axes both --step-mdeg 250
```

**Launch an MCP server** (e.g. the orchestrator) for an MCP client to connect to:

```bash
OPTICS_SIMULATE=1 PM_SIMULATE=1 uv run python -m mcp_servers.orchestrator_server
```

Key simulation switches: `OPTICS_SIMULATE=1` (mirrors), `PM_SIMULATE=1` (power meter).
Omit them on the bench PC and set the hardware env vars described in each server's
module docstring (`OPTICS_XA_SERIALS`, `PM_VISA_ADDRESS`, etc.).

---

## Data and models included

- **Trained twin:** `surrogate-NN/NN_parameter_data/fiber_coupling_model.pth`
  with `normalisation_parameters.npz`. Loaded by `surrogate_fiber_coupling.py` /
  `orchestrator_server.py` — no external download or Drive mount needed.
- **Sample inputs:** `surrogate-NN/sample_data/*.npz` (2-D angle maps on two mirrors).
- **Repeatability results:** `repeatability_logs/<run>/` — `samples.csv` plus
  `summary.json` and `manifest.json` carrying full provenance (protocol, device
  serial, git commit, host, timestamps).

---

## Stage repeatability study

A unidirectional-retreat repeatability study of the Thorlabs XA stage
(`tests/thorlabs_xa_repeatability_study.py`) was run for 25 cycles against 1 mm and
2 mm targets. The raw samples and full provenance (protocol, device serial, git
commit, host, timestamps) are logged under `repeatability_logs/` for inspection.

---

## Power meter (Thorlabs PM) and VISA — host notes

The power meter talks over **PyVISA**, which needs a **VISA runtime** on the host
(typically **NI-VISA** on Windows) installed separately from Python for USB/LAN
discovery and sessions. Without it, run with `PM_SIMULATE=1`. Set `PM_VISA_ADDRESS`
when auto-detection isn't appropriate.

> **Caution:** NI-VISA and the Thorlabs Optical Power Monitor app can interfere.
> After installing NI-VISA for PyVISA, do **not** reinstall the Thorlabs Optical
> Power Monitor to "fix" a setup — it can break PyVISA/NI-VISA communication.

---

## Architecture

MCP's client–server model maps cleanly onto the lab:

- **MCP servers** wrap each instrument and expose its controls as MCP tools.
- **Orchestrator (MCP client / run engine)** composes those tools, holds the
  alignment logic, and runs the optimize→command→measure loop.

The server-per-instrument split keeps the framework modular: additional devices
would join by adding a server, without changing the alignment orchestration.