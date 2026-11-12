# Dynamic Pricing Regulation for Public Shared Autonomous Vehicle Services

This repository contains the frozen computational record for a study of request-specific dynamic
pricing in a publicly managed shared autonomous vehicle service. The study formulates fare-setting
as a constrained Markov decision process in which the operator maximises economic return subject
to a long-run affordability requirement.

> **Status.** Submission package prepared for *Transportation Research Part B: Methodological*.
> The repository contains the source code, evaluation workflows, selected policy checkpoints,
> frozen manifests, result archives and SHA-256 provenance supporting the manuscript.

---

## Research problem

Request-specific pricing allows an on-demand transport operator to respond to variation in
demand, expected trip costs, modelled traveller valuations and vehicle availability. The same
flexibility can produce fare quotes above the benchmark associated with a prescribed share of
comparable travellers. The study examines whether dynamic pricing can improve economic
performance while satisfying a prespecified limit on the long-run mean exceedance above that
benchmark.

For the multiplier quoted at pricing decision `t`, the affordability cost and policy-level
requirement are

```text
c_t = max(0, m_t − m_t^aff)

J_C(π) = lim sup_{T→∞} (1/T) E_π[Σ_{t=0}^{T−1} c_t]

J_C(π) ≤ τ,    τ = 0.0025
```

Here, `m_t` is the executed fare multiplier and `m_t^aff` is the state-dependent affordability
benchmark for the assigned coverage target. The budget `τ` is expressed in fare-multiplier units.
The one-step cost is a hinge, so only the positive part of the exceedance contributes to the
long-run mean.

The executed action matters because the learned policy is stochastic. Training estimates the
constraint signal from the rollout mean of the raw sampled-action costs. Independent evaluation
re-estimates the same quantity on reserved annual seeds. A separate diagnostic evaluates cost at
the conditional mean action and isolates the contribution of stochastic action selection.

## Method

- **Two-critic proximal policy optimisation.** Separate critics estimate discounted economic
  return and unscaled affordability cost.
- **PI-Lagrangian constraint control.** The controller acts on the normalised constraint gap
  `J_C/τ − 1`. Rectification prevents negative penalty pressure when the constraint is inactive,
  and conditional integration limits integral overshoot.
- **Persistent demand response.** A demand-health state links accumulated price-out experiences
  to the future request process.
- **Exploration-premium decomposition.** The analysis compares sampled-action cost with cost at
  the conditional mean action to measure the affordability cost attributable to stochastic
  execution.
- **Independent confirmation.** Training, checkpoint selection and final annual evaluation use
  mutually disjoint seed sets.

## Evaluated policies and headline results

The study compares Reward-only dynamic pricing, three constrained policies trained at coverage
targets of 0.70, 0.80 and 0.90, and an optimised static fare multiplier of 1.05. The common-target
analysis evaluates every policy against the same 0.80 affordability benchmark. Reported
configuration means aggregate three independent training replications, each evaluated with five
independent annual seeds.

| Policy | Mean annual profit (€) | Affordability cost as % of budget | Feasible |
| --- | ---: | ---: | :---: |
| Reward-only | 297,407 | 1,977 | No |
| P70 | 297,500 | 1,461 | No |
| P80 | 297,551 | 85 | Yes |
| P90 | 272,882 | 0 | Yes |
| Static 1.05 | 247,828 | 34 | Yes |

The results support four principal findings.

1. **Moderate affordability protection has no statistically resolved annual-profit penalty.**
   Neither P70 nor P80 differs significantly from Reward-only in the paired annual comparisons.
   The most adverse lower confidence bound limits the profit reduction consistent with the
   intervals to 2.25% of Reward-only profit.
2. **P80 crosses the common compliance boundary with little change in estimated profit.** At the
   configuration means, P80 reduces common-target affordability cost by a factor of 23.3 relative
   to Reward-only and records slightly higher mean profit. The paired analysis does not
   statistically resolve that profit difference.
3. **The strictest tested policy trades profit for zero common-target exceedance.** P90 records
   zero cost in all fifteen final-seed evaluations under the common 0.80 benchmark. Its annual
   profit is 7.6% to 9.1% lower than Reward-only across the three training replications.
4. **Every learned dynamic pricing policy outperforms the optimised static fare in annual
   profit.** P80 earns 20.1% more than Static 1.05 at the configuration means. All paired
   confidence intervals comparing learned policies with the static benchmark exclude zero.

P80 and P90 constitute the non-dominated set among the tested configurations. P80 provides the
highest mean annual profit among policies that satisfy the common standard. P90 accepts lower
profit to eliminate measured common-target exceedance. These two observations describe the
efficient tested alternatives and do not estimate a continuous theoretical frontier.

At the assigned coverage targets, stochastic action selection accounts for 92.9% to 96.8% of
sampled-action affordability cost under P80 and 96.9% to 98.7% under P90, across the three
training replications. Compliance assessment must therefore evaluate executed stochastic quotes.
Evaluation based only on the conditional mean action understates the cost generated during
deployment.

Tables EC1–EC14 in `Supplementary_Material.docx` provide the complete replication-level results,
statistical comparisons and provenance records.

## Evaluation protocol

The final protocol separates policy development from independent confirmation.

| Stage | Purpose | Number of seeds |
| --- | --- | ---: |
| Training | Estimate policy parameters | 3 |
| Checkpoint selection | Choose one checkpoint for each policy instance | 5 |
| Independent confirmation | Estimate annual outcomes and test feasibility | 5 |
| Common-target accounting | Compare policies under the same 0.80 benchmark | Final annual seeds |

This reproduces the four stages recorded in Table EC11. Development seed roles used for
τ-calibration, preliminary-frontier exploration and smoke testing are mutually disjoint from the
above and are excluded from the final protocol.

Candidate checkpoints spanning 5 to 20 million training steps are evaluated with the five
checkpoint-selection seeds after a 36-month unmeasured burn-in. The prespecified reward and
feasibility rule selects one checkpoint for each policy instance. The selected checkpoint is then
frozen and evaluated with five reserved annual seeds. A confirmation outcome does not trigger
retrospective checkpoint replacement.

Assigned-target feasibility uses the unrounded one-sided 95% upper confidence bound based on the
0.95 quantile of Student's t-distribution with four degrees of freedom. Eight of the nine
constrained policy instances satisfy the affordability budget at full numerical precision. For
P90 under training seed 314159, the mean cost is 0.002461 and the upper confidence bound is
0.002500174. The bound exceeds the budget by 1.74 × 10⁻⁷, equivalent to 0.007% of the budget. The
prespecified rule classifies this instance as a failure at full precision.

Each final evaluation contains 12 measured chronological months after the 36-month burn-in. The
persistent endogenous state continues across the burn-in boundary. Reseeding at that boundary
separates the burn-in and measurement random-number substreams without discarding the mature
state produced during burn-in.

## Repository contents

```text
Supplementary_Material.docx      electronic companion, Tables EC1–EC14
src/                             simulator, configuration and learning implementation
scripts/training/                training entry point and Slurm launcher
scripts/validation/              runtime, demand-signal and persistence checks
workflows/static_fare/           static-fare grid evaluation and selection
workflows/checkpoint_selection/  held-out checkpoint selection
workflows/headline_evaluation/   independent mature-service annual evaluation
workflows/common_080/            common-0.80 accounting and documented patches
results/static_fare_selection/   static grid and frozen Static 1.05 selection record
results/*.tar.gz                 checkpoint-selection and annual evaluation archives
checkpoints/                     selected policy models and normalisation files
manifests/                       candidate and scenario manifests
environment/                     recorded Python and Conda environments
provenance/                      file manifest and SHA-256 checksums
```

`RESULT_COLUMN_MAPPING.md` maps manuscript measures to their archived columns and formulas.
`STATISTICAL_CONVENTIONS.md` records the confidence-bound calculations used in the evaluation.

## Reproduction

1. Create the recorded environment from `environment/`.
2. Obtain the three processed runtime inputs identified under **Data availability**, verify their
   hashes against Table EC12 and place them in `src/data/`.
3. Run the checks in `scripts/validation/`.
4. Train one policy for each configuration and training seed using `scripts/training/`.
5. Select checkpoints with `workflows/checkpoint_selection/`.
6. Reproduce the fixed-fare grid with `workflows/static_fare/`.
7. Run the final mature-service evaluation with `workflows/headline_evaluation/`.
8. Run the already patched common-target workflow in `workflows/common_080/`.

The frozen Slurm launchers and metadata retain the original HPC paths as provenance. Execution in
another environment requires updating the working directory, Python executable, model root and
output root and setting `PYTHONPATH` to `src/`.

The reported common-target results use `common_target_080_mature36/` in the common-target result
archive. The directory `common_target_080_failed_1992134/` contains the superseded pre-hotfix run
and contributes no reported result.

## Data availability

The three processed runtime inputs supporting the study are `hourly_demand.csv`,
`enriched_trip_requests.csv` and `zone_demand_summary.json`. They are deposited in the public
research-data record linked below. SHA-256 values in Table EC12 and `provenance/` identify the
exact versions used in the reported experiments.

**Data record:** `[INSERT PUBLIC DATA REPOSITORY URL BEFORE RELEASE]`

The present repository contains the code, checkpoints, manifests, workflows and frozen result
records required to audit the reported analysis. Re-execution additionally requires the three
processed input files from the linked data record.

## Integrity

`provenance/SHA256SUMS` records every file in the frozen release except the checksum file itself.
`provenance/FILE_MANIFEST.tsv` records the relative path, file size and SHA-256 hash of each
included file. Verification commands are provided in `provenance/VERIFY_ARCHIVE.txt`.

## Citation

Create a numbered release and archive it in a DOI-issuing repository before making the
computational record public. Replace the placeholders below with the complete manuscript author
list and the final DOI.

```bibtex
@software{zigah_cmdp_pricing_2026,
  author    = {Zigah, Elias and [COAUTHORS]},
  title     = {Computational Archive for Dynamic Pricing Regulation for Public Shared
               Autonomous Vehicle Services},
  year      = {2026},
  version   = {1.0.0},
  publisher = {Zenodo},
  doi       = {[DOI]},
  url       = {https://doi.org/[DOI]}
}
```

## Acknowledgements

The authors thank Ruter, the public transport authority for Oslo and Akershus, for providing the
operational data used in the simulation and for sharing practical insights into the design and
operation of on-demand shared autonomous vehicle services. Their feedback helped align the
modelling assumptions and policy questions with the operational context of public transport
provision. This research was conducted within the framework of the European Union's ULTIMO
project. The interpretations and conclusions remain those of the authors.


