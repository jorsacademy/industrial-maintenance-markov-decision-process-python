# Industrial Maintenance Markov Decision Process

A fully observed condition-based maintenance optimization example using Markov chains and an infinite-horizon discounted Markov Decision Process (MDP).

The asset condition is reviewed at regular discrete intervals:

```text
Healthy -> Degraded -> Critical -> Failed
```

At each review, the decision maker chooses one of:

```text
Operate
Minor maintenance
Replace
```

Failed equipment must be replaced.

## Objective

The model minimizes expected infinite-horizon discounted cost:

```text
E[ sum gamma^t * cost(S_t, A_t) ]
```

with default discount factor `gamma = 0.97`.

Costs are in stylized `$1,000` units per review period. They represent educational engineering assumptions, not calibrated plant accounting data.

## Operating degradation model

Default `Operate` transition matrix:

```text
             Next state
Current      Healthy  Degraded Critical Failed

Healthy       0.880    0.105    0.014   0.001
Degraded      0.000    0.730    0.220   0.050
Critical      0.000    0.000    0.520   0.480
Failed        0.000    0.000    0.000   1.000
```

Minor maintenance is imperfect and probabilistically improves condition. Replacement returns the asset to `Healthy` with probability `0.995` and `Degraded` with probability `0.005`.

Default state costs:

```text
Healthy       0
Degraded     35
Critical    160
Failed      850
```

Default action costs:

```text
Operate              0
Minor maintenance  120
Replace            420
```

## Independent exact solution methods

The same finite discounted MDP is solved with three independent methods:

1. value iteration;
2. policy iteration;
3. exhaustive enumeration of every valid stationary deterministic policy.

There are exactly:

```text
3 * 3 * 3 * 1 = 27
```

valid stationary deterministic policies in the default model.

For a fixed policy, the code solves:

```text
(I - gamma P_pi) V_pi = c_pi
```

exactly up to floating-point numerical precision.

## Validated default policy

All three exact methods return:

```text
Healthy   -> Operate
Degraded  -> MinorMaintenance
Critical  -> Replace
Failed    -> Replace
```

Default discounted values are approximately:

```text
Healthy      967.309
Degraded    1166.810
Critical    1519.257
Failed      2209.257
```

The Bellman residual is about:

```text
2.27e-13
```

## Monte Carlo cross-check

Monte Carlo simulation is not used to optimize the policy. It is an independent stochastic check of the exact policy value.

A representative 3,000-trajectory/state smoke run gives values close to the exact solution, within sampling error.

## Markov-chain diagnostics

Once a stationary policy is fixed, it induces a four-state Markov chain. The repository computes its stationary distribution and reports long-run average one-period cost and stationary failure probability.

For the default optimal policy, development diagnostics are approximately:

```text
V(Healthy)            967.309
stationary avg cost    30.077
stationary P(Failed)    0.111%
```

Two baseline policies are also evaluated.

### Run-to-failure

```text
Healthy   -> Operate
Degraded  -> Operate
Critical  -> Operate
Failed    -> Replace
```

Development diagnostics:

```text
V(Healthy)           3458.233
stationary avg cost   116.387
stationary P(Failed)    7.011%
```

### Replace only when Critical

```text
Healthy   -> Operate
Degraded  -> Operate
Critical  -> Replace
Failed    -> Replace
```

Development diagnostics:

```text
V(Healthy)           1976.287
stationary avg cost    64.709
stationary P(Failed)    1.360%
```

These are consequences of the declared model, not measured industrial failure rates.

## Sensitivity analysis

The optimal Critical-state decision changes when failure consequences change:

```text
Failed-state cost 400 -> Critical: MinorMaintenance
Failed-state cost 850 -> Critical: Replace
```

This shows why a maintenance policy depends on reliability and economic assumptions rather than on one universal threshold.

## Validation strategy

Regression tests verify:

- transition rows are valid probabilities;
- Failed-state action restrictions;
- a hand-checkable one-step Bellman backup;
- value iteration, policy iteration and exhaustive enumeration agree;
- all 27 stationary policies are enumerated;
- the optimal value dominates two intuitive baseline policies;
- stationary Markov-chain probabilities are valid and invariant;
- Monte Carlo runs are reproducible for a fixed seed;
- failure-cost sensitivity changes the Critical-state action.

## Run

```bash
python industrial_maintenance_mdp.py
```

Self-test:

```bash
python industrial_maintenance_mdp.py --self-test
```

Regression suite:

```bash
python -m unittest discover -s tests -v
```

Smaller CI-style stochastic run:

```bash
python industrial_maintenance_mdp.py \
  --monte-carlo-trajectories 3000 \
  --monte-carlo-horizon 220 \
  --seed 42
```

## Exactness versus modeling scope

The dynamic-programming solution is exact for the declared finite discounted MDP up to numerical tolerance.

That does **not** imply that the resulting policy is optimal for a real plant. Production use would require health-state definitions, transition probabilities, maintenance effectiveness and economic costs to be estimated and validated from actual operational data.

The model is fully observed. If equipment health were latent and only noisy sensor observations were available, a partially observable MDP (POMDP) would be a more appropriate formulation.
