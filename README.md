# SD-WAN Predictive NOC – Dataset Generator

Generates a synthetic 7-day SD-WAN network dataset (~50 000 rows) for training
a Predictive NOC ML model. Run `python run_all.py` to produce all CSV files.

## Quick Start

```bash
pip install numpy pandas
python run_all.py
```

Output CSVs are written to `./output/`.

---

## Pipeline Steps

| Step | File | Purpose |
|------|------|---------|
| 00 | `step_00_config.py` | Global constants, fault type list, severity sets |
| 01 | `step_01_topology.py` | Multi-site topology (P/PE/CE devices, tunnels, BGP peers) |
| 02 | `step_02_fault_schedule.py` | Non-overlapping fault schedule with fault_id + recovery_time |
| 03 | `step_03_telemetry.py` | Telemetry helpers with precursor ramp-up and severity distribution |
| 04 | `step_04_generate_rows.py` | Main simulation loop – emits all table rows |
| 05 | `step_05_save_csvs.py` | Saves 8 CSV files to `./output/` |
| 06 | `step_06_validate.py` | Comprehensive validation report |
| 07 | `step_07_split_dataset.py` | Time-based train/validation/test split (no shuffling) |
| 08 | `step_08_dataset_summary.py` | Builds `dataset_summary.json` (seed, fault counts, class distribution, feature list) |

---

## Fixes Applied (v2)

### Fix 1 – Target Labels
- `future_fault_type` contains only clean fault class names (no `recovery_*` prefix)
- Added `network_state` column: `normal | degrading | fault | recovery`
- Recovery rows: `future_fault_type=normal`, `network_state=recovery`

### Fix 2 – Precursor / Early Warning Behaviour
- Gradual ramp of latency, jitter, utilization, packet_loss before every fault
- Routing instability signals (BGP/OSPF events) grow during precursor phase
- Tunnel degradation metrics rise before tunnel_failure
- Model can learn: normal → warning signs → degradation → fault → recovery

### Fix 3 – time_to_impact_minutes
- Counts **down** to the fault (e.g. 10 → 5 → 0)
- Value = minutes until the fault enters its active phase
- During active phase: 0
- During recovery / normal: 0

### Fix 4 – Severity Distribution
- Targets: low ~50% | medium ~25% | high ~15% | critical ~10%
- Severity is now phase- and depth-aware (not just fault-type-based)
- Precursor → low/medium; active → high/critical; recovery → medium

### Fix 5 – Affected Service Realism
- `tunnel_failure` / `tunnel_rekey_storm` → VOICE, ERP, DATABASE
- `congestion` (datacenter) → DATABASE, ERP
- `link_failure` / `link_degradation` → VOICE, ERP, WEB
- `bgp_flap` / `bgp_hijack` → ERP, DATABASE

### Fix 6 – fault_id Column
- Added `fault_id` (e.g. `F001`, `F002`) to `ml_features.csv` and `fault_labels.csv`
- Groups all rows belonging to the same fault event

### Fix 7 – recovery_time_minutes Column
- Added `recovery_time_minutes` to `ml_features.csv` and `fault_labels.csv`
- Measures minutes the network takes to recover after a fault

### Fix 8 – Class Distribution Balance
- All 9 fault types guaranteed to appear
- Normal ratio: 30–40% | Fault/predictive: 60–70%

### Fix 9 – Time-Series Train/Test Split
- Subsampling preserves temporal order (no random shuffle)
- Use time-based split: first 70% train / next 15% val / last 15% test
- Prevents data leakage from future rows into training

---

## Fixes Applied (v4 — Round 3: Class Balance & Row Count)

The Round 2 dataset (v3) actually **overshot** the fault/predictive target —
71 fault scenarios at 30–90 min duration covered so much of the 7-day
timeline that normal traffic dropped to ~21%, well below the 30–40% target,
and the final file shipped only ~43,680 rows instead of the advertised
50,000. Round 3 fixes both issues.

### Fix 10 – Class Balance Correction
- A literal "120–150 scenarios at 30–90 min duration + 30–60 min precursor"
  target is **geometrically infeasible** for a 10,080-minute window with
  non-overlapping faults — those footprints alone exceed the entire
  timeline. The feasible, verified configuration is:
  - **65 total fault scenarios** (7–8 per type, all 9 types present)
  - **Precursor:** 10–30 min (early-warning / prediction window)
  - **Active duration:** 20–60 min (shorter than Round 2's 30–90 min, so
    more scenarios fit without normal traffic disappearing)
  - **Recovery:** 5–20 min (unchanged)
- Result: **normal ≈ 32%**, **fault/predictive ≈ 68%** — inside the
  30–40% / 60–70% validated target band.

### Fix 11 – Raw Row Count / Padding Bug
- Row generation sampled **4–5 devices per minute** (alternating, ~4.33
  avg), producing only ~43,680 raw rows over the 10,080-minute window —
  always short of `TARGET_SAMPLES=50,000`.
- The shortfall was "fixed" by tiling exact-duplicate rows to pad up to
  50,000 — but `step_05`'s `drop_duplicates()` step then removed every
  tiled duplicate, silently shipping ~43,680 rows instead of 50,000.
- Fix: sample a flat **5 devices/minute**, yielding 10,080 × 5 = **50,400**
  raw rows — already above target, so no padding or duplicate-tiling is
  ever needed. `ml_features.csv` now reliably contains exactly 50,000
  unique, chronologically-sorted rows with 0 duplicates.

---

## Train / Validation / Test Split (step_07_split_dataset.py)

A standalone script that splits `ml_features.csv` chronologically — it does
**not** train, fit, or evaluate any ML model; it only produces split CSVs.

```bash
python step_07_split_dataset.py
```

What it does:
1. Loads `ml_features.csv` from the output directory
2. Combines `date` + `time` into a single `timestamp` column (used only for
   sorting — it is not written to the output files)
3. Sorts rows chronologically (stable sort, **no random shuffling**)
4. Splits by position along the timeline:
   - **Train** – first 70%
   - **Validation** – next 15%
   - **Test** – last 15%
5. Saves `train.csv`, `validation.csv`, `test.csv` to the same directory as
   `ml_features.csv`
6. Prints row counts and timestamp ranges for each split
7. Runs validation checks: no overlapping timestamps between splits, train
   ends before validation begins, validation ends before test begins, and
   train + validation + test row counts sum to the original total

It never shuffles data, balances classes, drops rows, or modifies feature
values — every row from `ml_features.csv` ends up in exactly one of the
three output files, in its original form.

Already wired into `run_all.py` as **Step 07**, running automatically right
after validation.

---

## Dataset Summary (step_08_dataset_summary.py)

A standalone script that reads the already-generated `ml_features.csv` and
fault schedule and writes a single metadata file: **`dataset_summary.json`**.
It does not regenerate or modify any data.

```bash
python step_08_dataset_summary.py
```

`dataset_summary.json` contains:

| Section | Contents |
|---------|----------|
| `generation` | random seed, generation timestamp |
| `simulation` | duration in days/minutes, sample interval, start/end timestamps |
| `faults` | fault type list, total scenarios, scenarios per type (from the schedule), and row counts per `future_fault_type` label |
| `class_distribution` | `future_fault_type`, `network_state`, and `severity` breakdowns (count + %), plus a `normal_vs_fault` rollup |
| `features` | total column count and a `{name, dtype}` entry for every column in `ml_features.csv` |
| `dataset` | row count, target sample count, source filename, application/VRF lists |

Already wired into `run_all.py` as **Step 08**, running automatically right
after the train/validation/test split. Safe to re-run any time on its own —
it only reads existing output files.

---

## ML Features (ml_features.csv)

| Category | Columns |
|----------|---------|
| Time | date, time |
| Topology | node, role, site |
| Telemetry | utilization_pct, latency_ms, jitter_ms, packet_loss_pct, errors |
| Routing | bgp_events, ospf_events, route_changes |
| Tunnel | tunnel_state, tunnel_loss_pct, tunnel_latency_ms, rekey_count |
| Traffic | flow_bytes, flow_packets, application_type |
| App Impact | voice_latency_ms, video_packet_loss, erp_response_time_ms, database_latency_ms, application_traffic_drop_pct |
| NOC Context | severity, affected_nodes, affected_sites, affected_service, fault_id, network_state |
| Prediction Targets | future_fault_type, time_to_impact_minutes, recovery_time_minutes |

---

## Fault Types

`link_degradation` | `link_failure` | `bgp_flap` | `ospf_flap` | `congestion`
`route_leak` | `tunnel_failure` | `tunnel_rekey_storm` | `bgp_hijack`

---

## Supported ML Models

- RandomForest / XGBoost (tabular classification)
- LSTM / Transformer (time-series forecasting)
- Isolation Forest / Autoencoder (anomaly detection)
- Predictive NOC Copilot (natural language explanations)
