# Energy-Efficient Ground Station Selection for LEO Downlink (OCO-2) — RL Baselines

This repository contains a simulation environment for **LEO satellite downlink scheduling** and baseline policies (**Random / Greedy / DQN**) for **ground station selection** under **AoI (Age of Information)**, **queue dynamics**, and **energy cost** constraints.

> Core question: *Given time-varying visibility and distance-dependent transmission energy, which ground station should a satellite downlink to at each minute?*

---

## TL;DR

- **Environment**: minute-level LEO downlink simulator (Skyfield + TLE orbit propagation)
- **Actions**: choose **one** ground station (or no action) each step
- **Objective**: maximize a weighted reward capturing:
  - fresher information (AoI)
  - balanced queues / reduced overflow
  - energy sustainability (distance-dependent transmission cost)
- **Baselines**:
  - **Random policy** (visibility-aware random)
  - **Greedy policy** (choose visible station with minimum distance)
  - **DQN** (PER + n-step return)

---

## What’s inside

### Policies
- `random_policy.py`: visibility-aware random selection (among visible stations)
- `greedy_policy.py`: select the **closest visible** station (distance greedy)
- `agent.py`: DQN agent (Q-network, prioritized replay buffer, n-step return)

### Environment
- `env.py`: `OCO2Env` gym environment
  - Visibility map (`vis_map`) + distance matrix (`distance_all`) are precomputed
  - AoI and queue states evolve at each step
  - Energy update includes transmission energy and charging model

### Training / Evaluation scripts
- `night_train.py`: wall-clock time-limited DQN training (best/last checkpoints + curve PNG snapshots)
- `eval_only.py`: evaluation-only rollouts for Random/Greedy/DQN using saved checkpoints

---

## Problem setup

At each minute `t`:

- The satellite generates new data (LEO queue grows)
- The agent selects an action:
  - **NoAction** or **send to exactly one ground station**
- Transmission is only valid if the station is visible at that minute.
- Downlink updates:
  - LEO queue, GS queue (with capacity & overflow)
  - AoI at each ground station
  - Energy consumption based on distance-dependent transmit power (Friis/FSPL-inspired)

---

## Observation & Action

### Observation (55D or 65D)

`env.py` constructs a state vector including:

- time index (minute)
- AoI (LEO + per-ground-station)
- queue sizes (LEO + per-ground-station)
- current visibility flags (per station)
- current distance to each station
- normalized queue usage
- battery energy
- (optional) future line-of-sight features

### Action

- MultiBinary(10) in the environment (choose exactly one GS, or none)
- In the DQN code we map:
  - `action_idx == 0` → NoAction
  - `action_idx in [1..10]` → choose station `action_idx - 1`

---

## Quick start

### 1) Create environment & install dependencies

```bash
python -m venv .venv
# Windows
.\.venv\Scripts\activate
# Mac/Linux
# source .venv/bin/activate

pip install -r requirements.txt
