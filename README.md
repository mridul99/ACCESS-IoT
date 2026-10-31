# ACCESS-IoT

**A ground-truth dataset and benchmark for access control in healthcare IoT** — evaluation and reference-baseline code.

[![Dataset DOI](https://img.shields.io/badge/dataset-10.5281%2Fzenodo.23838631-blue)](https://doi.org/10.5281/zenodo.23838631)
Code: MIT · Data: CC BY 4.0

ACCESS-IoT is a semi-synthetic, time-ordered log of authorisation requests in a simulated hospital (300 users, 500 patients,
1,000 devices, six departments) with real IoMT network telemetry from CICIoMT2024 or WUSTL-EHMS-2020. Every request carries the
attributes a policy decision point sees at decision time and the ground truth that real logs lack:

- whether it is malicious, and which of seven insider and device threats produced it;
- whether a genuine clinical need existed, and whether the device path was compromised;
- the oracle decision (PERMIT / LIMITED / DENY / BREAK_GLASS), a reference decision under the benchmark's stated rules;
- a delayed outcome that is revealed only if the deciding mechanism's own action exposes it.

The **data** are on Zenodo: <https://doi.org/10.5281/zenodo.23838631>. This repository holds the documentation, the scoring
code and the reference baselines used in the paper. The data generator is not released; every build is fully documented and
reproducible from the released data, parameters and manifest.

## Repository contents

| Path | Contents |
|---|---|
| `notebooks/01_splits_views_validation.ipynb` | How the splits, views and history features were derived; fidelity (E1), validity (E2) and stability (E3) |
| `notebooks/02_baselines.ipynb` | Scoring code for tasks T1–T4 and all reference baselines (E4, E5, E7), Table 5 and Fig. 3 |
| `notebooks/03_transfer_xai.ipynb` | Telemetry ablation and transfer (E6), explanation ground truth (E8), Fig. 2d |
| `docs/TASKS.md` | Tasks, official evaluation protocols, metrics and the reward of the RL view |
| `requirements.txt` | Python packages |

## Getting the data

Download all files from Zenodo into one folder, e.g. `data/`, and unzip every zip there (the build zips, `results` and `docs`):

```
data/builds/main/requests.parquet
data/builds/main/data_dictionary.csv
data/builds/main/splits.parquet
data/builds/main/views/...
data/builds/main/features/history.parquet
data/builds/hard_fraud/...
...
```

Check the download by running `sha256sum -c MANIFEST.sha256` inside `data/`. The notebooks read the data from `data/` and write
tables and figures to `reports/`; to use other folders, set the environment variables `ACCESS_IOT_DATA` and `ACCESS_IOT_REPORTS`.
The paper's result tables are in `data/results/`; Notebook 3 can use them instead of re-running Notebook 2 (copy
`E4_T1_mean.csv` and `E7_per_threat_MGR.csv` to `reports/`).

| Build | Purpose |
|---|---|
| `main` | Reference build: seed 42, CICIoMT2024 telemetry (329,178 requests over 150 days, 3.54% malicious) |
| `sparse_feedback` | Half as many non-audit outcomes revealed |
| `noisy_benign` | More irregular legitimate behaviour |
| `hard_fraud` | Adds 12 covert emergency misusers whose claims copy genuine emergencies (official T2 build) |
| `no_telemetry` | ACCESS-IoT-Sim: no real network telemetry |
| `wustl_telemetry` | WUSTL-EHMS-2020 telemetry instead of CICIoMT2024 |
| `seed_101` … `seed_105` | Generator-seed replicates for stability |

**Inputs and labels.** `data_dictionary.csv` gives the visibility of every column. Only columns marked `request`, `pip`,
`telemetry` or `baseline` may be used as model inputs; `oracle` columns are labels and feedback columns are revealed later.

```python
import pandas as pd
d   = pd.read_parquet("data/builds/main/requests.parquet")
dic = pd.read_csv("data/builds/main/data_dictionary.csv")
X   = d[dic.loc[dic.Visibility.isin(["request", "pip", "telemetry", "baseline"]), "Column"].tolist()]   # model inputs
y   = d["OptimalDecision"]                                                                              # T1 label
sp  = pd.read_parquet("data/builds/main/splits.parquet")
train, val, test = (sp.temporal == "train"), (sp.temporal == "val"), (sp.temporal == "test")
```

## Tasks and official protocols (summary; details in `docs/TASKS.md`)

Thresholds and model choices are tuned on the validation days only (days 100–119); results are reported on the test days (120–149).

| Task | What is predicted | Official protocol | Metrics |
|---|---|---|---|
| T1 Access decision | Oracle decision of each request | Temporal split of `main`; generalisation on `loto_<threat>` and `dept_shift` | Macro-F1, malicious-grant rate, legitimate- and emergency-denial rates, delegation metrics, average reward |
| T2 Break-glass adjudication | Genuine vs fraudulent emergency claim where the static policy denies access | `actor_disjoint` split of `hard_fraud` (no test fraudster seen in training); covert and overt reported separately | AUROC with bootstrap 95% CI; fraud granted at the threshold honouring 99% of genuine emergencies |
| T3 Delegation risk | Will a delegation grant be misused? | Temporal split of `main`, test-period grants | AUROC, AUPRC |
| T4 Learning from delayed feedback | Decisions learned from rewards only | Chronological replay of the training days; outcomes delivered only if `AppliesIf` holds for the action taken; selection on validation reward; T1 metrics on test | As T1 |

Report the malicious-grant rate together with the legitimate- and emergency-denial rates (any one of them alone can be
minimised trivially), and results over several seeds or the seed replicates.

## Scoring your own method

Open `notebooks/02_baselines.ipynb`, run cells R0 and R1, then:

```python
c = Ctx("main")                          # loads data, splits, mask, features of one build
actions = my_method(c)                   # one action per request: 0 PERMIT, 1 LIMITED, 2 DENY, 3 BREAK_GLASS
print(t1_metrics(actions, c, c.te))      # T1 on the test period
print(t3_metrics(my_risk_score, c, c.te))  # T3: higher = more likely to be misused
```

For T2 use the actor-disjoint protocol of cell R7b (`t2_ci`) on the `hard_fraud` build. Actions outside the governance envelope
(`views/rl_admissible_mask`) are counted as envelope violations.

## Reference baselines

Static ABAC (with and without static break-glass), RBAC, ABAC policy mining (decision tree), CapBAC (with and without
audit-triggered revocation), trust-based access control (beta reputation), risk-adaptive, zero-trust re-evaluation, Isolation
Forest insider detection, gradient boosting (single request / with history features) and a delayed-reward neural contextual
bandit. They are simple, generic implementations meant as reference points, not tuned state of the art.

## Requirements

Python 3.10+, `pip install -r requirements.txt`. A GPU is recommended for XGBoost and the neural bandit; everything also runs on
the CPU, more slowly. Use a high-RAM machine (for example a Colab High-RAM runtime) for the baseline notebook.

## Citation

Please cite the dataset (and the article once published):

> Mishra Mridul Kailashchandra, U.P. Rao, P. Chauhan. ACCESS-IoT: A Ground-Truth Dataset and Benchmark for Access Control in
> Healthcare IoT [dataset]. Zenodo, v1.0, 2026. https://doi.org/10.5281/zenodo.23838631

Device telemetry is derived from CICIoMT2024 (Dadkhah et al., *Internet of Things* 28 (2024) 101351) and WUSTL-EHMS-2020
(Hady et al., *IEEE Access* 8 (2020) 106576–106584); please cite them as well.

## Licence

Code: MIT (see `LICENSE`). Data: CC BY 4.0, distributed on Zenodo.
