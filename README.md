# Condenser-Water Loop — Control Bench

A physics-based, browser-based transient simulator of a commercial building's condenser-water cooling loop — two evaporative cooling-tower cells and lead/standby condenser-water pumps. One self-contained HTML file: no build, no dependencies, no backend.

It's a **control and efficiency test bench**, not a data historian. The point is to rehearse the cooling-tower sequence of operations, try setpoints, compare pump/fan staging strategies, and see the energy cost — all against a model that behaves like the real plant, without touching the real plant. The simulation engine here is the same one used headless for offline analysis.

The model was built around **one specific building** and calibrated against its building-automation-system (BAS) field trends. It is fully **de-identified** — no site, owner, or tenant information is included — and every site-specific constant is exposed in one place, so it can be re-fit to a different plant (see [Adapting to another site](#adapting-to-another-site)).

![Control bench screenshot](docs/screenshot.png)

## What it does

- **Live animated process schematic.** Pipe color tracks water temperature, dash speed tracks each pipe's flow, and the fans/pumps spin at their actual modeled speed. Live values are shown at each real sensor and valve location.
- **The sequence runs the plant.** Tower staging, CWS low-limit protection, freeze logic, and economizer gating operate automatically — or take any actuator to **HAND** to override the sequence, *including over its safeties*, so you can drive the plant into failure modes on purpose.
- **KPI faceplate:** supply (CWS) / return (CWR) temperature, approach to wet-bulb, loop flow, heat rejection, and live electrical power draw (fans + pump, motor + VFD input) — the number you use to weigh staging and setpoint choices.
- **Everything is adjustable, live:** setpoints and sequencing thresholds, tower effectiveness knobs, hydraulic resistances, the DP-PID gains, ambient (OAT / wet-bulb), and a weather-driven building-load model.
- **Optional live weather.** Pick any US METAR/ICAO station (e.g. `KJFK`) or a global `lat,lon` pair, and the sim drives OAT + wet-bulb from real observations (NWS primary, Open-Meteo fallback). No location ships in the file — you choose it at runtime. This is the *only* network call; everything else runs locally and offline.
- **Alarm flags:** cavitation, low flow, low CWS, freeze, OAT lockout, over-HP, air ingestion, basin overflow, pump deadhead, basin low, and ΔT clamp.

## The physics

- **Pumps** — affinity laws on a digitized manufacturer head curve, with a part-load efficiency curve that falls off either side of BEP. Series/parallel hydraulic resistances are solved against the pump curve to find the operating flow.
- **Cooling tower** — open evaporative tower modeled with an effectiveness (ε-NTU) / air-side Merkel formulation using the wet-bulb enthalpy driving force; airflow follows fan affinity, with a fixed natural-draft floor when the fan is off.
- **Thermal transport** — first-order lags on the hot and cold legs, with the transport volumes pinned to the measured loop transients (not a geometric pipe trace).
- **Building load** — a bidirectional, economizer-gated load model: a 24/7 IT/CRAC floor, an occupancy-scheduled AHU compressor load that collapses to its economizing residual below the OAT/enthalpy limits, plus water-source heat pumps that add to or pull heat from the loop.
- **Response** — quasi-steady. Steady-state CWS tracks the field trend closely; transient fidelity is bounded (see limitations).

## Calibration and honesty

The equipment skeleton is the real installed plant: tonnage, pump horsepower, pipe sizes/lengths, and basin volume come from nameplate and engineering data. The **hydraulic resistances, the pump differential-pressure curve, the loop transport volumes, and the tower air-side cap were fit to BAS field trends** — pump DP, loop flow, supply/return temperature, and wet-bulb. Steady-state supply temperature reproduces the field trend to roughly **0.7 °F RMS**, and modeled loop flow lands within a few percent at the calibrated operating points.

This is a model. It is meant to be useful for *relative* comparisons (this staging vs. that one, this setpoint vs. that one) and for rehearsing sequences. Known limitations, stated plainly:

- **Wet-bulb sensitivity is under-modeled.** Across the wet-bulb range, the modeled tower effectiveness moves less than the field data shows, so the tower can read several °F optimistic at the low-wet-bulb corner. **It is not yet reliable for absolute free-cooling or economizer-hour predictions.** A tower-physics revision is planned, pending a fixed-fan wet-bulb sweep.
- **Loop load is treated as equal to tower heat rejection.** The compressor heat-of-rejection uplift (~×1.2) isn't applied yet, so rejection/tower duty read conservative relative to true condenser load.
- **Differential pressure is modeled as a function of flow.** That's right for this constant-flow building (no per-unit two-way valves), but it can't reproduce a measured DP-reset schedule.
- **The occupied-load floor was calibrated to a single cool, occupied day** — and checked against the same day, so that match is partly by construction. A second trend day is needed before trusting the building-load estimate on mild days.
- **Quasi-steady thermal response.** The building and return nodes carry no heat capacitance, so a sudden load step gives a slightly sharp return-temperature response. Steady-state behavior is the trustworthy part; transient accuracy is not yet characterized.

## Run it

It's a single static file.

- **Locally:** open `index.html` in any modern browser. That's it.
- **GitHub Pages:** Settings → Pages → deploy from branch, root folder. It serves directly (the file is named `index.html` for this reason).

No server, no build step, no install.

## Adapting to another site

The model is generic in form and specific in numbers. To point it at a different plant, edit the clearly-marked constants near the top of the `<script>` block in `index.html`:

- **`CFG`** — the live model state and all tunable constants in one object:
  - *Sequence / setpoints:* CWS setpoint, low-limit, bypass hold/close points, deadband, DP setpoint, pump minimum speed, GPM floor, freeze trip, OAT lockout.
  - *Tower:* `epsMax` (design effectiveness), `airCap` (air-side heat-rejection limit), fan exponents, `apprMin` (minimum approach), `minLeave` (cold-weather leaving-water floor).
  - *Hydraulics:* `kser` / `kcell` (series and per-cell resistances), `static` (open-tower lift), `kdp` (DP-curve coefficient), `cvByp` (bypass valve Cv).
  - *Building load:* `base24` (24/7 IT/CRAC floor), `occBase` (occupied AHU floor), `ahuPeak` (hot-day AHU max), `hpHeat` (heat-pump draw), and the economizer OAT/enthalpy limits.
- **`V` / `BASIN_GAL` / `FILL_GPM`** — loop transport volumes and basin geometry.
- **`FAN_KW`**, the pump BEP and efficiency curve, and the head-curve coefficients — equipment nameplate.
- **`SITE`** / the *Set location* control — live-weather station (empty by default; chosen at runtime).

To recalibrate against your own BAS: a couple of operating points at different pump speeds will anchor the hydraulics, and a fixed-fan wet-bulb sweep will anchor the tower. The inline comments document what each constant was fit to and which trend points pin it.

One thing that is **not** just a constant: the SVG process schematic and the sensor-tag positions are hand-laid for this plant's topology (two cells, this bypass arrangement, lead/standby pumps). If your plant differs — different cell count, header layout, or pump configuration — you'll edit the SVG and the render bindings, not just the numbers.

## Stack

Vanilla HTML / CSS / JavaScript. No frameworks, no build, no dependencies — a single file you can read top to bottom.

## License

MIT — see [LICENSE](LICENSE).

## Disclaimer

This is an engineering test and teaching tool. The constants were calibrated against one real plant, but the project contains no identifying site, owner, or tenant information. All outputs are model estimates and carry the limitations described above; do not use this as the basis for operating a real plant without independent engineering verification.
