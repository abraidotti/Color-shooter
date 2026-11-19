# color-shooter: Plan

## Milestones
- [ ] 1. Movement sandbox: first-person controller in a CSG test room; tune until it feels good
- [ ] 2. Polarity prototype: toggle LIGHT/DARK with cooldown; one standard hitscan weapon; target dummy
- [ ] 3. Connect two players: host/join by IP (listen server), spawn both, sync position and look
- [ ] 4. Authoritative combat: clients send fire input, the server raycasts and applies damage, death, respawn, polarity absorb
- [ ] 5. Rounds: countdown, fight, score, reset, first to N wins
- [ ] 6. Content: neutral piercing weapon, more weapons, 2 to 3 maps, HUD and menus
- [ ] 7. Deploy: headless Linux server export, AWS Lightsail, systemd; Windows client build

Playtest the polarity prototype (milestone 2 to 3) with a friend before building more weapons. If the toggle mind game is not fun with a basic pistol, rethink it.

## Setup
- Godot 4.x standard (GDScript build), pinned. Same version for client, server and CI.
- Git with Godot `.gitignore`. Git LFS only if large binaries appear.
- Godot editor for scenes and tuning; IDE + Claude Code for scripts (Godot LSP on port 6005).
- Local multiplayer: Debug > Run Multiple Instances, connect to 127.0.0.1.
- Tests: gdUnit4 or GUT. Lint: gdtoolkit.

## Movement ("middle ground")
Starting values, to be tuned by feel and held in a `MovementConfig` resource:
- Ground: max speed ~7 m/s, quick acceleration, strong friction.
- Jump: fixed height, coyote time ~0.1s, jump buffering.
- Air: lower acceleration than ground but steerable; cap applies only to input-driven velocity so jump and slide momentum carries.
- Slide: crouch while moving for a short speed boost with a cooldown; jumping out keeps momentum.
- Optional later: dash, small bhop bonus (exported toggles).
- Implementation: `CharacterBody3D` + `move_and_slide()`, simulate in `_physics_process`, camera as a child with mouse look.

## Polarity (central mechanic)
- Each player has polarity `LIGHT` or `DARK`, replicated by the server.
- A shot carries the shooter's polarity. Same polarity as the target: absorbed (no damage). Opposite polarity: damages the target.
- Absorbing fills a meter (use TBD, see open questions).
- Toggle cost, start simple: cooldown ~1 to 2s plus a short fire lockout after switching. Consider later: windup vulnerability (~0.2s), a toggle resource meter, absorb refilling that meter.
- Strong visual and audio language: clearly contrasting colors on model, weapon and tracers; distinct absorb effect and sound.
- Possible anti-stall counters, since both players can match polarity forever: neutral piercing weapon, flip shots, environment hazards.

## Weapons
Each weapon relates to polarity differently:
- Standard shot (pistol, rifle): carries your color, absorbed by matching target. Baseline.
- Neutral piercing shot: ignores polarity, low damage or slow fire. Counters stalling.
- Polarity-flip shot: hits and forces the target to switch for 1 to 2s.
- Absorb weapons: shield or beam that stores absorbed shots and fires them back with a bonus.
- Area weapons (grenade, shotgun): hurt all polarities, or dual-color at reduced damage.
- Dual-polarity weapon: alternates color every shot.
Hitscan is instant and leaves no time to react to polarity, so consider slower projectiles or a visible tracer with a few frames of delay.
Weapon acquisition options: random spawn pickups, one fixed weapon per polarity (toggling swaps gun), or a draft of 2 to 3 random picks between rounds.

## Networking
- Godot high-level multiplayer: ENet (UDP), `MultiplayerSpawner`, `MultiplayerSynchronizer`, `@rpc`.
- Server-authoritative; client-side prediction for the local player; interpolation for remote players; server-side hit detection. Lag compensation is a later extra.
- Start with 2 players, one small map, one weapon.
- Server order decides toggle versus hit races; a short toggle windup keeps this fair.

## Build and deploy
- Export presets committed: Linux dedicated server, Windows client.
- GitHub Actions: lint, tests, export both builds on push or tag (Godot container image).
- Hosting: AWS Lightsail Linux instance with IPv4 (starts about $5/month; the smallest plan has 512 MB RAM, so test headless memory use and size up if needed). Static IP included.
- Server runs under `systemd` with restart on crash.
- Firewall: open only the game UDP port plus SSH from my IP; key-based SSH. Use GitHub OIDC to AWS for automated deploys, not long-lived keys.
- Set an AWS billing alarm (about $15).
- Distribute Windows builds via GitHub releases. Version-check on connect.

## Art
Low-poly, flat-shaded, vertex-colored meshes generated in code (`SurfaceTool`, CSG for blockouts), shared resources or `MultiMeshInstance3D` for repeated props. CC0 packs (Kenney, Quaternius, Poly Haven) are a fallback. Keep one cohesive palette.

## Open questions (ask before deciding)
- Exact toggle cost model (cooldown only vs. windup vs. resource meter).
- What the absorb meter is used for.
- Weapon acquisition: pickups, per-polarity, or draft.
- Hitscan vs. slow projectiles for the baseline weapon.
- Whether to add dash or bhop on top of slide.
- Final game name (working repo name: color-shooter; fun candidate: Chroma Blast 3000).
