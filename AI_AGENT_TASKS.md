# AI Agent Implementation Tasks for PlatSplat

This checklist breaks the AI progression into concrete milestones you can execute in order:
1) Rule-based baseline
2) A* planner for static geometry
3) PPO RL for dynamic robustness
4) Optional behavior cloning -> PPO fine-tuning

---

## 0) Foundation (do this first for all agent types)

### 0.1 Build a unified agent interface
- [ ] Define an `Agent` interface with:
  - `reset(levelState)`
  - `act(observation) -> action`
  - `onStep(transition)` (optional for learning agents)
  - `onEpisodeEnd(summary)`
- [ ] Add an `AgentManager` that can load one agent implementation at a time.
- [ ] Add a UI toggle/dropdown to switch active agent: `Human | RuleBased | Planner | PPO | BC+PPO`.

### 0.2 Define action space
- [ ] Standardize action format as 4 booleans:
  - `left`, `right`, `jump`, `down`
- [ ] Enforce mutually exclusive horizontal controls (`left` XOR `right`) unless intentionally testing edge cases.
- [ ] Add a helper to convert policy outputs into valid game actions.

### 0.3 Define observation schema
- [ ] Create a compact observation object available each frame:
  - Player: position `(x,y)`, velocity `(vx,vy)`, grounded, wall-cling, alive.
  - Local tiles: fixed-size window around player (e.g., 15x11 chars encoded to ints).
  - Enemies: nearest `N` enemies with relative `(dx,dy)`, type, velocity.
  - Bullets: nearest `M` bullets with relative `(dx,dy)`, velocity.
  - Goal: relative vector to goal `(dx,dy)`.
- [ ] Version this schema (e.g., `obs_v1`) so later changes are traceable.

### 0.4 Add metrics and logging
- [ ] Episode metrics:
  - completion (0/1)
  - time-to-goal
  - max progress in x toward goal
  - deaths
  - damage/hits (if tracked)
- [ ] Add CSV/JSONL logging per episode for offline analysis.
- [ ] Add deterministic seed option for reproducible runs.

### 0.5 Headless evaluation harness
- [ ] Add a non-visual simulation mode to run many episodes quickly.
- [ ] CLI examples:
  - `npm run eval -- --agent=rule --episodes=100 --seed=42`
  - `npm run eval -- --agent=planner --episodes=100`
- [ ] Ensure human gameplay still works in normal render mode.

---

## 1) Rule-Based Baseline (prove interface + metrics)

### 1.1 Start with a simple heuristic controller
- [ ] Movement policy:
  - Move right by default.
  - Jump when obstacle ahead/gap ahead is detected in local tiles.
  - If wall-clinging and upward progress needed, trigger wall-jump rhythm.
- [ ] Threat response:
  - If bullet projected to intersect player soon, evade (jump or step back).
  - If runner enemy approaches and no platform escape, jump timing heuristic.

### 1.2 Implement feature detectors
- [ ] `isGapAhead(obs)`
- [ ] `isWallAhead(obs)`
- [ ] `isIncomingBullet(obs, horizonFrames)`
- [ ] `isEnemyThreat(obs, radius)`
- [ ] `shouldWallJump(obs)`

### 1.3 Baseline acceptance criteria
- [ ] Can complete simple/open test sections consistently.
- [ ] Survives longer than random policy by significant margin (e.g., +5x median episode length).
- [ ] Generates reproducible eval report with metrics.

Deliverable:
- `RuleBasedAgent` + evaluation report (`results/rule_based_baseline.json`).

---

## 2) Planner Agent (A*) for static level solve

### 2.1 Build a planning state model
- [ ] Define node state minimally as `(tileX, tileY, vxBucket, grounded, wallSide)`.
- [ ] Define transitions as simulated macro-actions over short horizon (e.g., 6-12 frames):
  - `run_left`, `run_right`, `short_jump_left`, `short_jump_right`, `wall_jump`.
- [ ] Use collision/physics-consistent forward simulation to generate neighbor states.

### 2.2 Graph search implementation
- [ ] Implement A* with:
  - `g(n)`: elapsed simulated frames
  - `h(n)`: heuristic distance to goal (tile distance + vertical penalty)
- [ ] Add closed-set hashing to avoid revisiting equivalent states.
- [ ] Add planning budget limits (max nodes/time) and fallback behaviors.

### 2.3 Plan execution layer
- [ ] Convert planned macro-actions to frame-by-frame button presses.
- [ ] Replan triggers:
  - Agent deviates from expected state.
  - Dynamic hazard invalidates planned path.
  - Timeout/stuck detection.

### 2.4 Planner acceptance criteria
- [ ] Solves static/no-enemy variant reliably (>95% success over seeds).
- [ ] Produces explainable path output (list of nodes/actions) for debugging.
- [ ] Gracefully degrades (fallback to safe movement) when budget exceeded.

Deliverable:
- `AStarPlannerAgent` + static-level benchmark report.

---

## 3) PPO RL Agent (robust dynamic behavior)

### 3.1 Environment wrapper
- [ ] Wrap game as Gym-like API:
  - `reset() -> obs`
  - `step(action) -> obs, reward, done, info`
- [ ] Ensure fixed-step simulation and deterministic seeding.
- [ ] Normalize/clip observations and rewards.

### 3.2 Reward design (first pass)
- [ ] Dense progress reward: positive for movement toward goal.
- [ ] Sparse completion bonus: large positive on reaching goal.
- [ ] Death penalty: significant negative.
- [ ] Time penalty: small negative per step to encourage speed.
- [ ] Optional shaping: survive projectile near-misses, maintain forward momentum.

### 3.3 PPO training pipeline
- [ ] Implement or integrate PPO trainer.
- [ ] Configure default hyperparameters:
  - rollout length
  - batch size
  - clip epsilon
  - value loss coefficient
  - entropy coefficient
  - learning rate schedule
- [ ] Save checkpoints + best-performing model by eval score.

### 3.4 Curriculum and robustness
- [ ] Curriculum stages:
  1. Static map, no enemies.
  2. Add runner enemies.
  3. Add shooter bullets.
  4. Full level dynamics.
- [ ] Domain randomization options:
  - enemy spawn timing jitter
  - small physics noise
- [ ] Periodic evaluation on fixed benchmark seeds.

### 3.5 PPO acceptance criteria
- [ ] Outperforms rule-based baseline on completion rate and median time.
- [ ] Maintains performance across unseen seeds.
- [ ] Training runs are reproducible from config + seed.

Deliverable:
- `PPOAgent` + training configs + model checkpoints + benchmark summary.

---

## 4) Optional: Behavior Cloning -> PPO Fine-Tune

### 4.1 Data collection
- [ ] Export successful human or scripted trajectories:
  - `(obs_t, action_t, done_t, return_t)`
- [ ] Record both high-quality and recovery scenarios (not only perfect runs).
- [ ] Split into train/val/test datasets.

### 4.2 Behavior cloning pretraining
- [ ] Train policy with supervised objective:
  - multi-label action prediction or categorical action class.
- [ ] Track imitation accuracy and rollout performance.
- [ ] Early stop/checkpoint on validation metrics.

### 4.3 PPO fine-tuning from BC initialization
- [ ] Initialize PPO policy weights from BC model.
- [ ] Lower initial learning rate, then schedule upward if needed.
- [ ] Compare sample efficiency vs PPO-from-scratch.

### 4.4 BC->PPO acceptance criteria
- [ ] Reaches target completion rate in fewer environment steps than PPO scratch.
- [ ] Avoids catastrophic forgetting of key movement skills.
- [ ] Final policy at least matches PPO scratch performance.

Deliverable:
- `BCPPOAgent` experiment report with ablation comparison.

---

## 5) Suggested folder structure

- [ ] Create and use a structure similar to:

```
/ai
  /core
    agent_interface.js
    observation_schema.js
    action_adapter.js
  /agents
    rule_based_agent.js
    astar_planner_agent.js
    ppo_agent.js
    bc_ppo_agent.js
  /training
    env_wrapper.js
    ppo_train.js
    bc_train.js
  /eval
    evaluate.js
    benchmarks.json
  /results
    *.json
```

---

## 6) Execution plan (practical order)

- [ ] Week 1: Foundation + rule-based baseline.
- [ ] Week 2: A* planner + static benchmark.
- [ ] Week 3-4: PPO environment wrapper + reward iteration + first converged model.
- [ ] Week 5: Optional BC pretrain + PPO fine-tune + comparative report.

---

## 7) Definition of done for the project

- [ ] One command can evaluate all agents on the same benchmark seeds.
- [ ] You have a single table comparing:
  - completion rate
  - median completion time
  - steps to first successful solve
  - robustness on unseen seeds
- [ ] Replay export works for each agent so behaviors can be visually inspected.
