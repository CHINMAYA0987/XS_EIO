<h1 align="center">SuperEIO</h1>

<p align="center">
  <em>Reproducing SuperEIO on a machine with none of its native dependencies — and beating the paper's own number on the sequence tested.</em>
</p>

<p align="center">
  <a href="https://arxiv.org/abs/2503.22963"><img src="https://img.shields.io/badge/arXiv-2503.22963-b31b1b" alt="arXiv"/></a>
  <a href="https://github.com/arclab-hku/SuperEIO"><img src="https://img.shields.io/badge/upstream-arclab--hku%2FSuperEIO-181717" alt="Upstream repo"/></a>
  <img src="https://img.shields.io/badge/Ubuntu-22.04-e95420" alt="Ubuntu 22.04"/>
  <img src="https://img.shields.io/badge/ROS2-Humble-22314e" alt="ROS2 Humble"/>
  <img src="https://img.shields.io/badge/Ceres-2.0-4c9a2a" alt="Ceres 2.0"/>
  <a href="RESULTS.md"><img src="https://img.shields.io/badge/write--up-RESULTS.md-a8611a" alt="RESULTS.md"/></a>
</p>

---

## Headline result

> After fixing a Ceres-version-related solver convergence bug, this reproduction reaches **0.32% Mean Position Error** on `boxes_translation` — against the paper's own **0.63%** for that sequence. The issue we faced while recording the original trajectory was the original repo used ROS1 and its native support, while we did all in ROS2 (main focus was on running the repo, visualization was not in the priority list).

| Run | Mean Position Error |
|---|---:|
| Paper (`boxes_translation`) | 0.63% |
| **This reproduction — after fix** | **0.32%** |
| This reproduction — before fix | 56.4% |

https://github.com/user-attachments/assets/026df61f-fab6-4271-9d55-4a827e0967cf

---

## What this is

A third-party reproduction run **without any of the repo's native dependencies**:

| | Paper's setup | This machine |
|---|---|---|
| **OS** | Ubuntu 20.04 | Ubuntu 22.04 |
| **Middleware** | ROS1 Noetic | ROS2 Humble |
| **Ceres** | 1.14 | 2.0 — 1.14 isn't packaged for 22.04 |
| **Dataset** | HKU sequences | public DAVIS240C — HKU mirrors weren't accessible |

The full narrative — environment, every issue hit and its fix, root-cause analysis, and all results — lives in **[`RESULTS.md`](RESULTS.md)**.

---

## Quickstart

```bash
# 1. One-time environment + build
#    (RoboStack ROS Noetic conda env, catkin workspace, patchelf fixes)
./scripts/build.sh

# 2. Get the dataset — public, no login required
#    See RESULTS.md §2 for why this sequence and not the paper's
wget https://download.ifi.uzh.ch/rpg/web/datasets/davis/boxes_translation.bag

# 3. Run + evaluate  (1.0 = real-time playback)
./scripts/run.sh /path/to/boxes_translation.bag 1.0

# 4. Regenerate plots from any run
python3 scripts/make_plots.py
```

---

## The key config change

The one substantive change behind the accuracy fix, in `config/240c/davis346.yaml`:

```diff
- max_solver_time:     0.04
- max_num_iterations:  8
+ max_solver_time:     0.5
+ max_num_iterations:  50
```

The shipped values were tuned for **Ceres 1.14**. This machine only has **Ceres 2.0** available, which needs more iterations and time to converge to the same quality per sliding-window optimization.

Full before/after data, including a separate real-time-throughput finding: [`RESULTS.md`](RESULTS.md) §5.

---

## Repository layout

| Path | Contents |
|---|---|
| [`RESULTS.md`](RESULTS.md) | Full write-up: environment, every build issue + fix, root-cause analysis, results |
| `scripts/` | `setup_toolchain_shims.sh`, `build.sh`, `run.sh`, plus `extract_gt.py` / `evaluate.py` / `make_plots.py` / `make_comparison_plot.py` |
| `plots/` | Figures and animation from the best run (0.32% MPE) |
| `plots_before_fix/` | The same figures from the original diverging run (56.4% MPE), for comparison |
| `results/` | Raw estimated trajectories (CSV) and ground truth (TUM) for the three key runs |
| `logs/` | Console log of the best run |
| `build_logs/` | All 22 build attempts, in order, showing every error and fix along the way |

---

## Also in here

### Sample event feature images

`plots/sample_event_features/` — the native detector's detection and tracking overlays, plus raw time-surface frames, captured live. See [`RESULTS.md`](RESULTS.md) §8.

### XFeat comparison

`plots/xfeat_keypoints.png`, `plots/xfeat_matches.png`, `scripts/xfeat_comparison.py` — pretrained, off-the-shelf XFeat run on the same event frames as the native detector: **~4× the keypoints** and **~6–7× the matches** per frame. See [`RESULTS.md`](RESULTS.md) §9.

### XFeat full-pipeline swap

`scripts/xfeat_feature_node.py`, `supereio_ba/launch/240c_xfeat.launch` — XFeat actually wired into the live VIO pipeline, not just compared against it, across six configurations. **None produced a trajectory.**

Diagnosed precisely with direct C++ instrumentation: the backend needs correspondences that are *simultaneously* long-baseline and high-count, which per-frame re-detection can't sustain. A seventh attempt — shrinking `WINDOW_SIZE` from 10 to 5, a compile-time constant — finally unblocked initialization for the first time, but the resulting trajectory diverges catastrophically (**2102% MPE**).

All attempts, full numbers, and next steps: [`RESULTS.md`](RESULTS.md) §10 (§10.7 for the last one).

---

## Citation

This repository reproduces work that is not mine. If you use SuperEIO itself, cite the original paper:

```bibtex
@article{SuperEIO,
  title   = {SuperEIO: Self-Supervised Event Feature Learning for Event Inertial Odometry},
  author  = {Chen, Peiyu and Lin, Fuling and Guan, Weipeng and Lu, Peng},
  journal = {arXiv preprint arXiv:2503.22963},
  year    = {2025}
}
```

**Paper:** [arXiv:2503.22963](https://arxiv.org/abs/2503.22963) · **Official implementation:** [arclab-hku/SuperEIO](https://github.com/arclab-hku/SuperEIO)

## Acknowledgements

All credit for the method belongs to the SuperEIO authors (Adaptive Robotic Controls Lab, HKU). This repository contributes only an independent reproduction on a different toolchain, plus the diagnostic work documented in [`RESULTS.md`](RESULTS.md).

The DAVIS240C sequence used here comes from the [RPG Event Camera Dataset](https://rpg.ifi.uzh.ch/davis_data.html) (Mueggler et al., IJRR 2017).
