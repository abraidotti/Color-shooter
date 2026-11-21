# color-shooter (working title)

A multiplayer first-person arena shooter for 2 to 8 players (first tested 1v1) in **Godot 4.x (GDScript)**. The central mechanic is Ikaruga-style **polarity**: each player is LIGHT or DARK. A shot is absorbed by a same-polarity target and damages an opposite-polarity one. Movement is a "middle ground": simple and forgiving, with slide and light air control. The tone is silly, so the game should not take itself too seriously.

Full design, milestones and open questions: @docs/PLAN.md

## Design principle: mind games first
The fun is outwitting the other player. When designing or tuning anything, prefer features that create readable information, meaningful commitment, bluffing and counterplay over raw stats or reflexes. Every mechanic should give players something to read and something to hide. Telegraph dangerous actions (so opponents can react), make commitments costly (so choices matter), and avoid options that are strictly best or that remove decisions.

## Architecture rules
- **Server-authoritative.** The server owns positions, health, polarity, hit resolution and round state. Clients send inputs only and never decide hits.
- **Shared movement code.** One movement function is used by the client (prediction) and the server (simulation). Keep input reading separate from movement math.
- **Polarity resolution happens on the server** by comparing shooter and target polarity at the moment of impact.
- **One codebase, two modes.** The same project runs as a client or, with `--server`, as a headless dedicated server (`godot --headless --path . -- --server`). The game must also run as listen-server (host + client) for local testing.
- Design for **up to 8 players** (`MAX_PLAYERS = 8`), including odd counts. Never hardcode 2 anywhere: use a dynamic player list, 8+ spawn points per map, and team/score logic that works for any N (1 to 8). The first milestones are still tested with 2.
- Client and server must check a build version on connect and refuse mismatches.

## Conventions
- Godot version is pinned (record it in `docs/PLAN.md` when chosen). Do not use APIs from other versions.
- Tunable values (movement, toggle cooldown, weapon stats) live in `.tres` Resources (`MovementConfig`, `WeaponData`), not in code.
- Typed GDScript. Format and lint with `gdformat` and `gdlint` (gdtoolkit).
- Prefer procedural low-poly meshes (`SurfaceTool`) with vertex colors and flat shading. Avoid textures.
- Commit after each milestone. Keep commits small.

## Layout
- `player/` controller, camera, weapon holder
- `weapons/` weapon resources and fire logic
- `net/` host/join, lobby, spawning
- `game/` round state machine, scoring
- `maps/` arenas
- `gen/` procedural mesh generators
- `docs/` design docs

## Working agreement
- Ask before making a decision listed under "Open questions" in `docs/PLAN.md`.
- Do not start a milestone before the previous one is playable.
- Backlog items in `docs/PLAN.md` wait until the MVP playtest gate passes.
- Reuse only MIT-licensed code; never copy GPL/AGPL code. Record borrowed code in `CREDITS.md`.
