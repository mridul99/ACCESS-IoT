# ACCESS-IoT

**A ground-truth dataset and benchmark for access control in healthcare IoT** — evaluation and reference-baseline code.

[![Dataset DOI](https://img.shields.io/badge/dataset-10.5281%2Fzenodo.24307413-blue)](https://doi.org/10.5281/zenodo.24307413)
Code: MIT · Data: CC BY 4.0

ACCESS-IoT is a semi-synthetic, time-ordered log of authorisation requests in a simulated hospital (300 users, 500 patients,
1,000 devices, six departments) with real IoMT network telemetry from CICIoMT2024 or WUSTL-EHMS-2020. Every request carries the
attributes a policy decision point sees at decision time and the ground truth that real logs lack:

- whether it is malicious, and which of seven insider and device threats produced it;
- whether a genuine clinical need existed, and whether the device path was compromised;
- the oracle decision (PERMIT / LIMITED / DENY / BREAK_GLASS), a reference decision under the benchmark's stated rules;
- a delayed outcome that is revealed only if the deciding mechanism's own action exposes it.

"Ground truth" here means ground truth with respect to the simulator: the labels are exact records of the simulator's state and
of the published oracle rules, not judgements adjudicated by clinicians or auditors on real events.

**Threat model.** Three adversary classes (malicious insiders, hijacked accounts, compromised devices) and seven threats:
gradual snooping, account takeover, collusion, delegation misuse, privilege escalation, emergency (break-glass) misuse and
device tampering. The security metric is the malicious-grant rate (share of malicious requests granted in any form), reported
per threat and together with the legitimate-denial and emergency-denial rates.

The **data** are on Zenodo: <https://doi.org/10.5281/zenodo.24307413> (version 1.1, all 11 builds; version 1.0,
<https://doi.org/10.5281/zenodo.23838631>, lacks the `hard_fraud` build). This repository holds the documentation, the scoring
code and the reference baselines used in the paper. The data generator is not released: the builds are fixed, their parameters
are documented in each `build_config.json`, their files can be verified against the manifest, and every reported result can be
reproduced from the released data and this code, but the data cannot be regenerated or new configurations created.

## Repository contents

| Path | Contents |
|---|---|
| `notebooks/01_splits_views_validation.ipynb` | How the splits, views and history features were derived; fidelity (E1), validity (E2) and stability (E3) |
| `notebooks/02_baselines.ipynb` | Scoring code for tasks T1–T4, the first thirteen reference baselines on every build (E4, E5, E7), and the leave-one-threat-out and department-shift results |
| `notebooks/03_transfer_xai.ipynb` | Telemetry ablation and transfer (E6), explanation ground truth (E8), Fig. 2d |
| `notebooks/04_additional_baselines_heldout_actors.ipynb` | Additional baselines (ReBAC, two insider detectors, GRU sequence model, off-policy bandit), ten seeds with 95% confidence intervals, and the actor-disjoint protocol for all threats and tasks; the paper's main results table and per-threat figures |
| `docs/TASKS.md` | Tasks, official evaluation protocols, metrics and the reward of the RL view |
| `requirements.txt` | Python packages |

## Getting the data

Download all files of the Zenodo record into one folder, e.g. `data/`, and run `python unpack.py` there (Python 3, no packages
needed). Every file on Zenodo is at most 40 MB, so the builds come as zip parts (`ACCESS-IoT-v1.1_main_1_of_N.zip` …) and the
few files larger than 40 MB even after compression are cut into pieces; `unpack.py` extracts every part, joins the pieces listed
in `SPLIT_FILES.json` and checks every file against `MANIFEST.sha256`. You may download only the parts of the builds you need;
files of other builds are then reported as missing, not as errors. The result is:

```
data/builds/main/requests.parquet
data/builds/main/data_dictionary.csv
data/builds/main/splits.parquet
data/builds/main/views/...
data/builds/main/features/history.parquet
data/builds/hard_fraud/...
...
data/results/          data/results/rev1/          data/docs/
```

Without Python, extract every zip into `data/`, join each split file with `cat` (Linux, macOS) or `copy /b` (Windows) as
described in the dataset's README, and check with `sha256sum -c MANIFEST.sha256` inside `data/`. The notebooks read the data from `data/` and write
tables and figures to `reports/`; to use other folders, set the environment variables `ACCESS_IOT_DATA` and `ACCESS_IOT_REPORTS`.
The paper's result tables are in `data/results/` (`rev1/`: the revised article's tables, which supersede the overlapping
v1.0 tables); Notebook 3 can use them instead of re-running Notebook 2 (copy
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
| T1 Access decision | Oracle decision of each request | Temporal split of `main` and the all-threat actor-disjoint split (below); generalisation on `loto_<threat>` and `dept_shift` | Macro-F1, malicious-grant rate, legitimate- and emergency-denial rates, delegation metrics, average reward |
| T2 Break-glass adjudication | Genuine vs fraudulent emergency claim where the static policy denies access | Actor-disjoint split of `hard_fraud` (no test fraudster seen in training); covert and overt reported separately | AUROC with 95% CI; fraud granted at the threshold honouring 99% of genuine emergencies |
| T3 Delegation risk | Will a delegation grant be misused? | Test-period grants of `main`, temporal and actor-disjoint splits | AUROC, AUPRC |
| T4 Learning from delayed feedback | Decisions learned from rewards only | Chronological replay of the training days; outcomes delivered only if `AppliesIf` holds for the action taken; selection on validation reward; T1 metrics on test | As T1 |

Report the malicious-grant rate together with the legitimate- and emergency-denial rates (any one of them alone can be
minimised trivially), the number of malicious test requests per threat, and results over at least ten seeds with confidence
intervals.

**Held-out actors (`actor_disjoint_all`).** The temporal split tests defence against adversaries whose earlier behaviour was
available for training. The actor-disjoint split tests defence against new adversaries using the same threats. For each
threat, the malicious actors (the device for device tampering, the delegatee for a delegation grant, the requesting user
otherwise) are divided into two halves: malicious requests of one half appear only in training and validation, those of the
other half only in test, and all other malicious requests are excluded. Benign requests are unchanged, and feedback events
about excluded requests are withheld. The split is computed from released columns by `actor_split` and `set_protocol` in
Notebook 4; for emergency misuse it equals the `actor_disjoint` column of `splits.parquet`.

## Scoring your own method

Open `notebooks/02_baselines.ipynb`, run cells R0 and R1, then:

```python
c = Ctx("main")                          # loads data, splits, mask, features of one build
actions = my_method(c)                   # one action per request: 0 PERMIT, 1 LIMITED, 2 DENY, 3 BREAK_GLASS
print(t1_metrics(actions, c, c.te))      # T1 on the test period
print(t3_metrics(my_risk_score, c, c.te))  # T3: higher = more likely to be misused
```

To score on held-out actors, and for T2, use Notebook 4: run cells R0–R5, S0 and S1, then

```python
c = Ctx("hard_fraud"); set_protocol(c, "actor_disjoint_all")   # sets c.tr, c.va, c.tem and filters the feedback events
actions, score = my_method(c)                                  # train on c.tr, tune on c.va
print(t1_metrics(actions, c, c.tem), per_threat(actions, c, c.tem))
print(t2_detail(score, c, c.tem, boot=1000))                   # higher score = more likely genuine; overt and covert separately
```

Actions outside the governance envelope (`views/rl_admissible_mask`) are counted as envelope violations.

## Reference baselines

Eighteen simple, generic implementations, meant as reference points and not as tuned state of the art:

| Group | Baselines |
|---|---|
| Rule-based | Static ABAC (with and without static break-glass), RBAC, ReBAC, ABAC policy mining (decision tree), CapBAC, zero-trust re-evaluation |
| Learning from revealed outcomes | CapBAC with audit-triggered revocation, trust-based access control (beta reputation) |
| Label-free risk and insider detection | Risk-adaptive; insider detection by Isolation Forest, per-user deviation and autoencoder |
| Supervised | Gradient boosting (single request / with history features), GRU sequence model |
| Reward-only | Delayed-reward neural contextual bandit, off-policy bandit (doubly robust) |

Malicious-grant rate on the test period of `main` (learned methods: mean of ten seeds):

| Baseline | Temporal split | Held-out actors |
|---|---|---|
| CapBAC | 0.963 | 0.967 |
| ReBAC | 0.820 | 0.823 |
| Static ABAC | 0.671 | 0.647 |
| Insider detection (user deviation) | 0.403 | 0.429 |
| Off-policy bandit | 0.263 | 0.393 |
| Delayed-reward bandit | 0.234 | 0.389 |
| GRU sequence model | 0.234 | 0.478 |
| Gradient boosting, with history | 0.209 | 0.518 |

On `hard_fraud`, no baseline separates covert break-glass fraud by held-out fraudsters (AUROC at most 0.68). The full tables,
with confidence intervals, are in the paper and are reproduced by Notebook 4.

## Requirements

Python 3.10+, `pip install -r requirements.txt`. A GPU is recommended for XGBoost and the neural bandit; everything also runs on
the CPU, more slowly. Use a high-RAM machine (for example a Colab High-RAM runtime) for the baseline notebooks. Notebook 4
saves every finished run to `reports/rev1/runs/`, so an interrupted run resumes; set `FAST = True` in cell S0 for a quick check.

## Citation

Please cite the dataset (and the article once published):

> Mishra Mridul Kailashchandra, U.P. Rao, P. Chauhan. ACCESS-IoT: A Ground-Truth Dataset and Benchmark for Access Control in
> Healthcare IoT [dataset]. Zenodo, v1.1, 2026. https://doi.org/10.5281/zenodo.24307413

Device telemetry is derived from CICIoMT2024 (Dadkhah et al., *Internet of Things* 28 (2024) 101351) and WUSTL-EHMS-2020
(Hady et al., *IEEE Access* 8 (2020) 106576–106584); please cite them as well.

## Licence

Code: MIT (see `LICENSE`). Data: CC BY 4.0, distributed on Zenodo.
