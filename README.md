# Bayesian Enriched Triplet–POD for uncertainty-aware real-time tumour tracking in cine-MRI

Analysis code for the manuscript *"Bayesian Enriched Triplet–POD for uncertainty-aware
real-time tumour tracking in cine-MRI"* (submitted to *Physics in Medicine & Biology*).
The pipeline reproduces every table and figure in the paper from the public
TrackRAD2025 challenge data.

> **Status / scope.** This repository contains the TrackRAD2025 centroid-level and
> dense optical-flow analyses that back the manuscript's results. It does **not**
> redistribute the raw challenge images or masks — see [Data](#data). All quantitative
> claims in the paper come from the CSVs in `tables/` produced by this code.

---

## What the pipeline does

The entry point is a single script, **`trackrad2025_bayes.py`** (exported from a
Colab/Kaggle notebook; run top to bottom). It has two branches:

- **Branch 1 — Dense optical-flow POD.** GPU Horn–Schunck apparent in-plane flow fields,
  classical displacement POD vs. enriched Triplet–POD, and cohort-level spectral, rank,
  and spontaneous-event analysis on a set of representative cases (`MAX_DENSE_CASES = 8`).
- **Branch 2 — Centroid Bayesian trajectory analysis.** Manual-mask centroid trajectories,
  Bayesian displacement and triplet coefficient models (linear-Gaussian / Kalman observed
  state), the truncation-residual calibration correction, reliability gating, risk–coverage,
  a χ² event monitor, an empirical-Bayes Student-*t* predictive, and bootstrap confidence
  intervals — over all 88 labelled cases.

A final section measures per-step online runtime and exports everything as a zip.

---

## Installation

```bash
git clone https://github.com/<user>/<repo>.git
cd <repo>
python -m venv .venv && source .venv/bin/activate     # optional
pip install -r requirements.txt
# Install the torch build matching your hardware from https://pytorch.org
```

A CUDA GPU is **recommended** for Branch 1 (the Horn–Schunck solve runs 300 iterations
per frame). Branch 2 is CPU-only and fast. With no GPU the code falls back to CPU
automatically (`DEVICE = "cuda" if torch.cuda.is_available() else "cpu"`).

---

## Data

The script pulls three Kaggle datasets via `kagglehub` (a Kaggle account / token is
required):

| Kaggle dataset | Role |
|---|---|
| `tanyamushininga/trackrad101` | TrackRAD2025 frames + target masks |
| `tanyamushininga/clean-trackrad` | cleaned/indexed variant |
| `tanyamushininga/4dxcat` | 4D-XCAT supplementary phantom |

Expected on-disk layout (per case): `images/<case_id>_frames.mha` and
`targets/<case_id>_labels.mha`. The original TrackRAD2025 data are available from the
challenge (arXiv:2503.19119, <https://trackrad2025.grand-challenge.org/>); please obtain
them under the challenge's own license rather than from a re-host.

---

## Running

The script was written for the Kaggle/Colab filesystem and **hardcodes input/output
paths** at the top. Before running outside that environment, edit the configuration block:

```python
TRACKRAD_ROOT = Path("/kaggle/input/.../TrackRad2025")   # -> your local data root
OUTDIR        = Path("/kaggle/working/trackrad_two")      # -> your output dir
```

(The reliability-gate and runtime sections write to two additional folders,
`trackrad_rejection_gate_fixed/` and `trackrad_online_runtime/`; repoint those too.)

Then:

```bash
python trackrad2025_bayes.py
```

Outputs land under `OUTDIR/`:
- `figures/` — manuscript figures (PNG, 300–350 dpi)
- `tables/`  — CSV tables (the numbers in the paper)
- `cached_dense_hs_fields/` — cached Horn–Schunck `.npz` fields (so Branch 1 is re-runnable
  without recomputing flow)

### Key configuration parameters

| Parameter | Value | Meaning |
|---|---|---|
| `TRAIN_FRAC` | `0.50` | temporal train/test split per case |
| `CROP_MARGIN` | `60` | tumour-centred crop margin (px) |
| `KAPPA_ACC` | `0.25` | acceleration-block down-weight κ in the triplet state |
| `HS_ALPHA`, `HS_N_ITER` | `15.0`, `300` | Horn–Schunck smoothness weight and iterations |
| `EVENT_Z_THRESHOLD` | `3.0` | robust-z threshold for the independent event label |
| `K_LIST` | `[1,2,3,4,5,8,10,12,15,20]` | POD ranks evaluated |
| `MAX_DENSE_CASES` | `8` | dense-branch cohort size |
| seed | `123` | NumPy RNG seed (bootstrap, etc.) |

---

## Reproducing each result

The script section that produces each manuscript artifact, and the file it writes:

| Manuscript item | Code section | Output file(s) |
|---|---|---|
| Prediction (Table 1) | 6 / 6C | `tables/paper_table1_prediction.csv` |
| Calibration (Table 2) | 6 | `tables/centroid_coverage.csv` |
| Reliability & gate (Table 3) | 6C / gate section | `tables/paper_table3_gate.csv`, `…_case_level.csv`, `…_frame_level.csv`, `…_summary.csv`, `tables/centroid_risk_coverage.csv` |
| Event AUROC (Table 4) | 6 / 6C | `tables/paper_table4_event_auroc.csv`, `tables/centroid_event_metrics.csv` |
| Dense reduced-order (Table 5) | 5 / 5B | `tables/dense_pod_rank_metrics_aggregate.csv`, `tables/dense_pod_case_summary.csv`, `tables/dense_pod_spectral_aggregate.csv` |
| Empirical-Bayes Student-*t* | 6C | `tables/paper_eb_studentt.csv` |
| Runtime table | runtime section | `trackrad_online_runtime/tables/…` |
| Preprocessing / centroid (Fig.) | 2–3 | `figures/fig1_…`, `figures/fig2_…` |
| Horn–Schunck example (Fig.) | 4B | `figures/fig3_hs_apparent_motion_example.png` |
| Dense POD spectrum / rank (Figs.) | 5B | `figures/fig4_…`, `figures/fig5_…`, `figures/fig6_…`, `figures/fig6b_…` |
| Prediction error (Fig.) | 6B | `figures/fig7_centroid_prediction_error.png` |
| Calibration (Fig.) | 6B | `figures/fig8_centroid_calibration.png` |
| Risk–coverage (Fig.) | 6B | `figures/fig9_risk_coverage.png` |

---

## Notes for re-users

- The file is a notebook export run **once, top to bottom**. A few small helpers
  (`boot_ci`, `mask_centroid`, `causal_derivatives`, …) are intentionally redefined in
  later sections so each block is self-contained; this is harmless when run in order.
- Branch 1 caches flow fields to `cached_dense_hs_fields/`; delete that folder to force
  recomputation, or set `overwrite=True` in `compute_and_cache_dense_hs`.
- Refactoring the hardcoded paths into CLI arguments / environment variables is the
  obvious first improvement for use outside Kaggle.

---

## Citation

If you use this code, please cite the paper and the archived release:

```bibtex
@article{mushininga_triplet_pod,
  title   = {Bayesian Enriched Triplet--POD for uncertainty-aware real-time
             tumour tracking in cine-MRI},
  author  = {Mushininga, Keith and Hbid, Moulay L. and El Hamidi, Abdallah
             and Denis de Senneville, Bruno},
  journal = {Physics in Medicine \& Biology},
  year    = {2026},
  note    = {Code: https://doi.org/10.5281/zenodo.XXXXXXX}
}
```

(Replace the DOI once Zenodo mints it for your tagged GitHub release — see the project
notes on creating a citable archive.)

## License

Released under the terms in [`LICENSE`](LICENSE). TrackRAD2025 data are governed by the
challenge's own license and are not redistributed here.
