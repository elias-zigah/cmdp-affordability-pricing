# Dynamic pricing under affordability regulation

Constrained deep reinforcement learning for fare setting in a publicly managed shared autonomous
vehicle service. The operator maximises profit subject to an affordability standard that must
hold in expectation over the deployed stochastic policy.

> **Status.** Manuscript under preparation for submission to *Transportation Research Part B: Methodological*.
> This repository currently holds the reproducibility record below. Code and the replication
> archive are planned for release following acceptance, subject to data-sharing permissions.

---

## The problem

A public operator running an autonomous ride service faces a fare decision with two sides. Raising
fares protects margin and shortens waits by suppressing demand. It also prices out the passengers
the service was publicly mandated to carry, and in a market with memory it erodes the demand base
that makes the service viable in the first place.

Formulated as a constrained Markov decision process, the operator maximises discounted profit
subject to a bound on the expected per-decision affordability cost:

```
c(s, a) = max(0, m − m̂)                        one-step cost on a single fare quote
J_C(π)  = E_{s ~ d_π,  a ~ π(·|s)} [ c(s, a) ]   constrained quantity
J_C(π) ≤ τ                                      regulatory budget
```

where `m` is the applied fare multiplier, `m̂` an affordability benchmark derived from the
willingness-to-pay distribution at a chosen coverage probability, `d_π` the state distribution
induced by the policy, and `τ` a budget in fare-multiplier units. Cost is a hinge, so only
exceedance is charged, and the expectation is over actions sampled from the stochastic policy
because a deployed policy issues sampled quotes.

Three quantities are distinct and are kept distinct throughout. `J_C(π)` is the constrained
object. During training it is estimated by the rollout mean of `c` over sampled actions, and that
estimate is the signal the dual controller acts on. At evaluation it is re-estimated on
independent seeds and tested by a one-sided upper confidence bound. A separate diagnostic
evaluates `c` at the conditional mean action, which is not the constrained quantity but isolates
systematic overpricing from exploration (see below).

## Method

- **Two-critic PPO.** Separate value heads for reward and constraint cost.
- **Lagrangian dual** on the constraint, with the multiplier driven by a normalised rectified PI
  controller: the controller acts on the relative gap `J_C/τ − 1`, the proportional term is
  rectified at zero so an inactive constraint cannot push the multiplier negative, and
  conditional integration prevents windup.
- **Equilibrium multiplier audit.** A frozen-checkpoint sweep estimates the equilibrium dual
  variable independently of training, by three separate zero-crossing criteria. The estimates
  agree and predict the converged operating point out of sample, which makes the dual value a
  measured quantity rather than a training artefact.
- **Exploration premium decomposition.** Constraint cost is reported both for sampled actions and
  for the conditional mean action. The difference isolates how much of the budget stochastic
  execution consumes, as distinct from systematic overpricing by the learned fare rule.

## Headline results

Five policies compared: unconstrained dynamic pricing, three constrained policies trained at
affordability coverage targets of 0.70, 0.80 and 0.90, and an optimised static fare of 1.05. All
evaluated against a common 0.80 affordability benchmark, averaged over three independent training
replications and five independent annual evaluation seeds.

| Policy | Annual profit (€) | Budget used (%) | Feasible |
| --- | ---: | ---: | --- |
| Unconstrained | 297,407 | 1,977 | No |
| Coverage 0.70 | 297,500 | 1,461 | No |
| Coverage 0.80 | 297,551 | 85 | Yes |
| Coverage 0.90 | 272,882 | 0 | Yes |
| Static fare 1.05 | 247,828 | 34 | Yes |

Three findings.

**The affordability constraint at 0.80 coverage is close to free.** The constrained policy earns
within the noise of unconstrained pricing while using 85% of a budget the unconstrained policy
overruns nearly twentyfold. Paired annual-profit differences against unconstrained pricing contain
zero in all six policy-by-replication comparisons at the 0.70 and 0.80 targets, that is, two
coverage targets across three independent training replications.

**Tightening to 0.90 coverage is not free.** Annual profit falls by €22,600 to €27,400 depending
on training replication, and every interval excludes zero. The policy prices at the affordability
benchmark almost everywhere, consuming none of the budget.

**Dynamic pricing beats the best static fare under every affordability setting.** Margins of
€48,400 to €51,500 per year for unconstrained and loosely constrained policies, €24,100 to €25,800
at the tightest setting. Every interval excludes zero.

A fourth result is methodological. Stochastic execution, not the fare rule, consumes most of the
affordability budget: the exploration premium accounts for 50% to 99% of sampled-action constraint
cost across policy instances. A constrained policy that looks tight on the sampled measure may be
pricing well inside the benchmark at its conditional mean. Reporting only the scalar sampled cost
hides this.

## Evaluation protocol

The claim a constrained-RL paper has to support is that a policy satisfies its constraint on data
it was not selected on. The protocol separates three stages with mutually disjoint seed sets,
verified in code before the evaluator will run.

| Stage | Purpose | Seeds |
| --- | --- | ---: |
| Training | Estimate policy parameters | 3 |
| Checkpoint selection | Choose one checkpoint per policy instance | 5 |
| Independent confirmation | Estimate annual outcomes, certify feasibility | 5 |

Candidate checkpoints spanning 5 to 20 million training steps are scored on the selection seeds
under a prespecified reward-and-feasibility rule. The selected checkpoint is then frozen and
evaluated on seeds never previously used. A confirmation outcome does not trigger retrospective
checkpoint replacement, which is what would turn confirmation back into selection.

Feasibility is certified by a one-sided 95% upper confidence bound on constraint cost lying at or
below the budget. One of fifteen policy-by-seed cells exceeds its bound by 1.7 × 10⁻⁷, around
0.007% of the budget, and is reported as a numerical boundary case rather than rounded into a pass.

Each evaluation runs 12 measured months after a 36-month unmeasured burn-in, so policies are
assessed on a mature demand state rather than on the transient. The endogenous demand state is
carried across the burn-in boundary while the random-number generator is reseeded there, which
separates the burn-in and measurement substreams without discarding the state the burn-in built.

## Reproducibility

The archive binds results to the code and inputs that produced them by SHA-256 over source files,
processed data, scenario manifests, selection reports and result archives. Reproduction runs in
six steps: verify provenance, select checkpoints on the selection seeds, run annual confirmation
on the confirmation seeds, verify accounting and measurement-boundary invariants, run common-target
accounting, and construct the reported aggregates.

The archive exists and the hashes are recorded in the manuscript's supplementary material. Its
structure, for release:

```
environment/   pinned interpreter and dependency lock
data/          processed inputs and checksums; access instructions for restricted sources
src/           simulator, two-critic PPO, dual controller, evaluators
manifests/     frozen scenario manifests and design fingerprints
results/       seed-level results, policy summaries, paired comparisons
logs/          run metadata and completion checks
```

**Data availability.** Simulator inputs were supplied by Ruter through the Oslo ULTIMO pilot and
are not redistributable. The repository ships processed-input checksums and access instructions so
a replication can verify it is working from identical data.

## Citation

```bibtex
@unpublished{zigah_cmdp_pricing,
  author = {Zigah, Elias},
  title  = {Constrained Markov decision process for dynamic pricing
            under long-run affordability requirements},
  note   = {Under review, Transportation Research Part B: Methodological},
  year   = {2026}
}
```

