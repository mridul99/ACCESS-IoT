# Tasks, protocols and metrics

Actions: `PERMIT` (0), `LIMITED` (1, served with restrictions pending verification), `DENY` (2), `BREAK_GLASS` (3).

## Oracle decision

The oracle is a reference decision under the benchmark's rules, not a clinically adjudicated judgement. It is derived from
the simulator's latent state and the static policy:

1. a request permitted by the static ABAC policy → `PERMIT`;
2. a request denied by it → `BREAK_GLASS` if it is an access request by a clinician with role permission, an emergency claim
   and a genuine clinical need; otherwise `DENY`;
3. malicious requests → `DENY`;
4. a benign request through a compromised device that would otherwise be granted → `LIMITED`;
5. approving a delegation to a future misuser → `DENY`.

**Governance envelope** (`views/rl_admissible_mask`): `PERMIT`/`LIMITED` only if static ABAC permits; `BREAK_GLASS` (or
`LIMITED`) on a policy denial only for an emergency claim by a clinician with a passed capability check, never from an
object-layer device; `DENY` is always admissible.

## Splits (`splits.parquet`)

| Column | Definition |
|---|---|
| `temporal` | train days 0–99, validation 100–119, test 120–149 |
| `loto_<threat>` | as `temporal`, with every training row of that threat removed (`excluded`) |
| `dept_shift` | train/validation on four departments, test on the other two (Cardiology, Radiology) |
| `actor_disjoint` | emergency-misuse actors split alternately into seen and held-out; seen actors' fraud only in train/validation, held-out actors' fraud only in test |

Tune on `val` rows only; report on `test` rows.

## T1 — access decision

Predict the oracle decision of each test request (temporal split; generalisation on `loto_*` and `dept_shift`).

| Metric | Definition |
|---|---|
| Macro-F1 | over the four actions |
| MGR, malicious-grant rate | share of malicious requests granted in any form (PERMIT, LIMITED or BREAK_GLASS) |
| LDR, legitimate-denial rate | share of oracle-PERMIT requests denied |
| EDR, emergency-denial rate | share of oracle-BREAK_GLASS requests denied |
| BG-Mis | share of malicious emergency claims in break-glass contexts granted by BREAK_GLASS or LIMITED |
| DG-Mis | share of later-misused delegation grants that were approved |
| DU-Exp | share of malicious requests under a delegation that were granted |
| Envelope violations | actions outside the governance envelope |
| Average reward | mean immediate reward (below) |

## T2 — break-glass adjudication

Score genuine (oracle `BREAK_GLASS`) against fraudulent (malicious) emergency claims made where the static policy denies
access. **Official protocol:** `actor_disjoint` split of the `hard_fraud` build, so that no test fraudster appears in training
or validation; report covert (`ActorTag == "CovertEmergencyMisuser"`) and overt fraudsters separately. The temporal split is a
reference only, because the same fraudsters appear in training and test.
Metrics: AUROC with a bootstrap 95% interval (1,000 resamples) and the share of fraudulent claims granted at the threshold that
honours 99% of genuine emergencies (higher score = more likely genuine).

## T3 — delegation risk

Score each test-period delegation grant for later misuse (`DelegationOutcome == "Misused"`) on the temporal split.
Metrics: AUROC, AUPRC (base rate 25.2% in the main build).

## T4 — learning from delayed, action-dependent feedback

The mechanism replays the training period chronologically, day by day, without oracle labels. It receives the immediate reward
of each action, except for deferred decisions (`BREAK_GLASS`, `LIMITED` on an emergency claim, approval of a delegation grant).
Their reward arrives with the outcome in `views/feedback_events.parquet` on its `FeedbackTime`, and only if `AppliesIf` holds
for the action taken (`always`, `break_glass_granted`, `delegation_approved`, `request_granted`); otherwise a neutral no-news
reward of +0.5 arrives after 21 days. The model is selected by validation reward and evaluated on the test period with the T1
metrics. The `sparse_feedback` build tests the same protocol with fewer revelations.

## Reward of the RL view

Immediate reward r(a, a\*, m, ζ), with ζ = 1 + s/4 for resource sensitivity s (2–4) and m = malicious:

| Action a | Oracle a\* | Reward |
|---|---|---|
| any | a\* = a | +1 |
| LIMITED | PERMIT or BREAK_GLASS | +0.5 |
| PERMIT or BREAK_GLASS | LIMITED | −ζ |
| PERMIT or BREAK_GLASS | DENY | −5ζ if m, otherwise −2ζ |
| LIMITED | DENY | −2.5ζ if m, otherwise −ζ |
| DENY | PERMIT or LIMITED | −1 |
| DENY | BREAK_GLASS | −4 |

Deferred rewards on revelation: misuse revealed −5ζ (−2.5ζ for LIMITED); legitimate use confirmed +1 (+0.5 for LIMITED);
compromised path detected +1 for LIMITED, −ζ otherwise; no news within 21 days +0.5.
