# PlatSplat
Platformer, testing ai players


## AI roadmap
See `AI_AGENT_TASKS.md` for an actionable implementation checklist for rule-based, A*, PPO, and BC→PPO agents.


## In-browser AI agents
- `Platformer_H.html` now includes `PPO` and `BC+PPO` options in the Agent dropdown.
- `PPO` performs online policy optimization with a clipped objective and value baseline.
- `BC+PPO` warm-starts the policy using behavior cloning from scripted teachers, then fine-tunes with PPO updates.
- The `Planner (A*)` agent now runs in a static-mode simulation (enemy/bullet updates paused) so it can follow geometry-only plans.
