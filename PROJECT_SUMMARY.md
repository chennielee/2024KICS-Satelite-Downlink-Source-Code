# Project Summary — Energy-Efficient Ground Station Selection for LEO Satellite Downlink (RL)

## 1) Problem
A LEO satellite generates new observation data over time and must decide **which one of 10 ground stations** (or “no action”) to downlink to at each minute.  
The decision must balance:
- **Freshness / timeliness** of information (Age of Information; AoI)
- **Queue dynamics** at the satellite and at each ground station (overflow / imbalance)
- **Energy cost** for communication (distance-based transmit power)
- **Time-varying visibility** (Line-of-Sight constraints)

This problem is naturally sequential and constrained: a locally “obvious” choice (e.g., nearest station) may not be optimal if it causes queue imbalance, energy depletion, or missed future opportunities under changing visibility.

---

## 2) Why it is challenging (and why interpretability matters)
During training/evaluation, I observed **counter-intuitive routing behavior**: the learned policy sometimes chooses a ground station that appears suboptimal by a simple heuristic (e.g., not the nearest / not the immediately best-looking option).

This raised a core question for me:
> If a policy’s decision is not explainable, it is difficult to debug whether it is (a) a real long-horizon trade-off, (b) a reward shaping artifact, or (c) instability/overfitting.

However, standard DQN compresses complex network state into scalar Q-values, making “why” hard to verify without brute-force probing across many cases.

---

## 3) Environment & Formulation (high-level)
### State (observation)
The environment state is a vector summarizing the satellite-network condition:
- time index (minute)
- LEO AoI and GS AoI (per station)
- LEO queue and GS queues (per station)
- visibility (LoS) per station at current time
- distance-to-station per station (used for energy modeling)
- normalized queue usage and current battery energy  
(optionally includes a future LoS feature)

### Action
A binary selection over ground stations (1 station at most per step).  
(Implemented as a single discrete index 0..10 internally, then mapped to a MultiBinary action vector.)

### Reward (multi-objective)
Reward is shaped to encourage:
- lower AoI / fresher data at ground stations  
- balanced and non-overflowing queues (penalize imbalance and overflow)  
- energy-aware transmission (distance-based communication energy + battery preservation)  
- penalties for invalid choices (no LoS, insufficient energy, etc.)

---

## 4) Baselines & Evaluation Protocol
To make results interpretable and reproducible, I compare against:
- **Random (visibility-aware)**: choose a random visible ground station (or no-op if none visible)
- **Greedy**: among visible stations, choose the nearest (minimum distance)

**Evaluation is run with epsilon fixed to 0** (pure exploitation) for the DQN policy, so the comparison reflects the learned policy—not exploration noise.

Outputs include per-episode returns and step-level logs for AoI/energy/queue.

---

## 5) Reproducibility (how to run)
### Training (time-limited, checkpointed)
- `night_train.py`
  - saves:
    - `checkpoints/dqn_agent_best.pt` (best episodic return)
    - `checkpoints/dqn_agent_last.pt` (last checkpoint)
    - periodic training curve PNGs for reporting

### Evaluation (policy-only)
- `eval_only.py` (recommended)
  - runs N episodes each for Random / Greedy / DQN(best or last)
  - writes:
    - `{policy}_log.csv` (episode returns)
    - `{policy}_steps_log.csv` (AoI + energy per step)
    - `{policy}_queue_log.csv` (queue per step)
  - generates PNG plots from CSV for reporting

---

## 6) Key artifact I used to motivate interpretability
The main “why” moment was not that the model failed—but that it sometimes made decisions that were hard to justify from any single metric (distance, AoI, or energy alone).  
This observation motivated my interest in structured / interpretable decision-making (e.g., decomposing decisions into higher-level goals vs. low-level actions).

---

## 7) Notes / Known limitations
- Because the environment is simulation-based (TLE + visibility), measured performance depends on reward shaping and environment settings.
- If a policy appears to “prefer distant” stations, it can be:
  - a legitimate long-horizon trade-off (future LoS / queue equalization)
  - a reward artifact (penalty scales, overflow penalty, energy penalty factor)
  - training instability (off-policy + function approximation)

The evaluation scripts are designed to separate these factors via AoI/energy/queue logs, not return alone.
