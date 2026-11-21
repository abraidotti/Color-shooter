# color-shooter: Plan

## Design principle: mind games first
Mind games (reading and bluffing opponents, as in Ikaruga) are the central design principle. Checklist for any new mechanic, weapon, map or mode:
- Information: can the opponent read something (a color, a glint, a sound)?
- Commitment: does acting cost something (cooldown, lockout, camo break) so choices matter?
- Bluff: can a player fake or hide intent (toggling mid-charge, hiding in matching terrain)?
- Counterplay: is there always a smart answer, and no strictly best option?
Prefer designs that pass this checklist over raw stats or pure reflex.

## MVP scope
MVP = 1v1, one small map, polarity with a toggle cost, one standard weapon, rounds, a minimal lobby, and basic sounds. Everything else is in the backlog below and waits for the playtest gate.

## Milestones
- [ ] 1. Movement sandbox: first-person controller in a CSG test room; tune until it feels good
- [ ] 2. Polarity prototype: toggle LIGHT/DARK with cooldown; one standard hitscan weapon; target dummy
- [ ] 3. Connect two players: host/join by IP (listen server), spawn both, sync position and look
- [ ] 4. Minimal deploy and first playtest: headless Linux server on AWS Lightsail (systemd), Windows client build, play with the friend
- [ ] 5. Authoritative combat: clients send fire input, the server raycasts and applies damage, death, respawn, polarity absorb
- [ ] 6. Rounds: countdown, fight, score, reset, first to N wins
- [ ] 7. MVP polish: lobby, main menu, HUD, generated sound effects, one map
- [ ] **Playtest gate:** play the MVP with the friend. Only if polarity mind games are fun, start the backlog.

Playtest the polarity prototype (milestones 2 to 4) with a friend before building more weapons. If the toggle mind game is not fun with a basic pistol, rethink it.

## Backlog (post-MVP, in rough order; each must pass the mind-games checklist)
1. Map color zones and camouflage
2. Color affinity (pre-match choice)
3. Neutral piercing weapon and polarity-flip shot (anti-stall)
4. Scoped sniper, with lag compensation
5. More weapons and weapon acquisition (pickups or draft)
6. Game modes for 3 to 8 players (free-for-all, king of the hill, winner-stays, hunter rounds)
7. More maps, room codes, optional third polarity

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

## Map color and camouflage
- Map surfaces/zones carry a polarity tag (LIGHT, DARK or NEUTRAL); the visual color comes from the same tag.
- If a player's polarity matches the zone they are in, they fade into it (shader dissolve). The server decides camo state and replicates it; camo is visual only and never changes hit detection.
- Moving fast, shooting or toggling breaks camo briefly, so nobody can camp invisibly. Keep camo partial (shimmer or faint outline) and give audio cues so hidden players can still be hunted.
- Polarity now has two jobs: combat (same color absorbs) and hiding (match the terrain); the best color for each can differ.
- Map ideas: split light/dark halves with a contested middle, checkerboard floors, zones that flip on a timer or switch, neutral cover.

## Weapons
Each weapon relates to polarity differently:
- Standard shot (pistol, rifle): carries your color, absorbed by matching target. Baseline.
- Neutral piercing shot: ignores polarity, low damage or slow fire. Counters stalling.
- Polarity-flip shot: hits and forces the target to switch for 1 to 2s.
- Absorb weapons: shield or beam that stores absorbed shots and fires them back with a bonus.
- Area weapons (grenade, shotgun): hurt all polarities, or dual-color at reduced damage.
- Dual-polarity weapon: alternates color every shot.
- **Scoped sniper rifle (required):** zoomed scope (FOV lerp on the camera, scope overlay, scaled mouse sensitivity, slight sway), slow fire. Its shot carries the shooter's polarity, so a target who matches color absorbs it, which makes toggling the core counterplay. Telegraphing keeps it fair: a charge-up or lock-on with a visible laser or scope glint in the shooter's color and an audio cue, so the target has a reaction window and the sniper can bluff by toggling mid-charge. Camo interaction: holding still while scoped keeps camo; firing breaks it and the glint can reveal the shooter. Variant: a neutral (polarity-ignoring) anti-stall sniper that is slower and louder. Sniper hits make lag compensation (rewinding positions on the server) worth adding earlier, and maps need cover against long sightlines.
**Fire modes (hitscan now, projectile later):** weapons are data (`WeaponData`) with a fire mode, and the server resolves every shot through one function that takes a shot record (shooter, origin, direction, timestamp, shooter polarity at fire time, weapon id). MVP implements only the `HITSCAN` resolver. A later `PROJECTILE` resolver spawns server-simulated projectiles (swept ray checks each tick to avoid tunneling) that carry the shooter's polarity from the moment of firing while the target's polarity is checked at impact, so toggling while a projectile is in flight becomes a new mind game. Clients only render projectiles.
Hitscan is instant and leaves no time to react to polarity, so consider slower projectiles or a visible tracer with a few frames of delay.
Weapon acquisition options: random spawn pickups, one fixed weapon per polarity (toggling swaps gun), or a draft of 2 to 3 random picks between rounds.

## Networking
- Godot high-level multiplayer: ENet (UDP), `MultiplayerSpawner`, `MultiplayerSynchronizer`, `@rpc`.
- Server-authoritative; client-side prediction for the local player; interpolation for remote players; server-side hit detection. Lag compensation is a later extra.
- Start testing with 2 players, one small map, one weapon, but build for up to 8 (`MAX_PLAYERS = 8`): dynamic spawning, no hardcoded player counts, per-player (not per-pair) scoring.
- Server order decides toggle versus hit races; a short toggle windup keeps this fair.
- Server hardening: validate every RPC argument and sender, rate-limit RPCs, cap message sizes, sanitize player names, ignore impossible inputs (speed, fire rate, polarity changes during cooldown), never trust client-reported hits, and keep no secrets in the client build.

## Build and deploy
- Export presets committed: Linux dedicated server, Windows client.
- GitHub Actions: lint, tests, export both builds on push or tag (Godot container image).
- Hosting: AWS Lightsail Linux instance with IPv4 (starts about $5/month; the smallest plan has 512 MB RAM, so test headless memory use and size up if needed). Static IP included.
- Server runs under `systemd` with restart on crash.
- Firewall: open only the game UDP port plus SSH from my IP; key-based SSH. Use GitHub OIDC to AWS for automated deploys, not long-lived keys.
- Set an AWS billing alarm (about $15).
- Distribute Windows builds via GitHub releases. Version-check on connect.

## Pre-match choice: color affinity
- Players pick a **color affinity** in the lobby. Locked at Ready; optionally changeable between rounds. The server validates it and applies it at spawn; its values live in a Resource (like `WeaponData`) that the shared movement and toggle code reads on both client and server.
- Affinity is a permanent lean toward one polarity: toggling into your affinity color is faster and out of it slower; camo is stronger in your color; you start each round in it. Optional "Chameleon" affinity: better camo, weaker shots.
- Team balance: start with no restrictions. If some choices prove too strong, give them point costs and equalize team budgets (smaller team gets extra points). Odd counts: the lone player on the short side gets a bonus or stat boost. Free-for-all ignores team rules.

## Player count and odd numbers (target: 1 to 8)
- No two-player assumptions: spawn points >= 8, dynamic player list, generic scoring.
- Two polarities means same-colored players cannot hurt each other with standard shots, so color acts as a temporary team and toggling is switching sides. Odd counts work naturally (uneven splits), but all-same-color stalemates need the neutral piercing weapon, flip shots, or hazards.
- Odd-friendly mode ideas: free-for-all, king of the hill / control point, winner-stays duel queue (others watch or harass), random-asymmetric rounds (one "hunter" with a buff vs the rest), last-one-standing.
- Team modes with uneven teams: scale health or damage for the smaller team, and fill with a bot if desired.
- Possible variant: three polarities in a rock-paper-scissors cycle (more complexity; decide later).

## Accessibility
- The game is color-coded, so never rely on color alone. LIGHT and DARK differ in brightness (not just hue); also give each polarity a distinct shape or pattern on the player model, weapon and tracers, a clear HUD icon, and a distinct sound.
- Camo zones and map polarity tags use patterns or geometry cues in addition to color.
- Offer a colorblind-safe palette option and adjustable polarity colors in Settings.
- Settings: mouse sensitivity, FOV, remappable keys, toggle-or-hold options (slide, scope), volume sliders.

## Audio
- Generate sound effects in code at startup (client only; skip when `DisplayServer.get_name() == "headless"`): synthesize samples (pitch sweeps, square/sine waves, noise, decay envelopes) into `AudioStreamWAV` resources via `PackedByteArray.encode_s16`, cache them, and reuse. Optional later: `AudioStreamGenerator` for continuous or adaptive sounds.
- Polarity has a distinct timbre: LIGHT is high and bright (sine-ish), DARK is low and buzzy (square-ish), so players can tell sides by ear.
- Core sounds: shot (fast downward sweep + noise), absorb (rising chime), toggle (noise whoosh), hit, death, footsteps and slide (audio cues for hunting hidden players), camo break.
- Use `AudioStreamPlayer3D` for world sounds and `AudioStreamPlayer` for UI. Buses: Master, SFX, Music, each with a volume slider.
- Sounds are cosmetic: the server sends events (fired, absorbed, died) and clients play them locally.
- Imported WAV files from a DAW are fine too; crunchy 8-bit sounds suit the silly tone.

## Player experience and playtesting
- Distribution: Windows client build (.exe + .pck, zipped) attached to a GitHub Release by CI; friend downloads, unzips, runs. Unsigned, so expect a SmartScreen warning. The same project exports as the headless Linux server. Client and server check build versions on connect and tell the player to download the latest build on mismatch. No web build (browsers cannot use ENet UDP).
- Main menu: name field, Play Online (connect to the server), Host and Join by IP (LAN/dev), Settings (mouse sensitivity, FOV, volume, keybinds), Quit.
- Lobby: player list with colors, Ready button, map and mode selection (by host or first player), game starts when all are ready.
- In game: scoreboard (Tab), pause menu with settings, rematch screen after a match.
- Matchmaking: none needed at first. Start with one always-on dedicated server on Lightsail and a shared lobby with ready-up. Add room codes (a tiny lobby service on the same server) only if several simultaneous games are needed. Steam (GodotSteam) is a later option for friends lists, invites and NAT punch-through.
- Playtest flow: two local instances while developing; deploy the server to Lightsail once milestone 3 works; send the friend the build and play with Discord for voice; include a debug overlay (ping, FPS, server tick) and log files to diagnose issues.

## License and reuse
- Repo license is undecided. Reuse only MIT-licensed code (for example the Liblast Framework, OpenFPS, the True First Person Multiplayer template), preferably as a reference. Do not copy GPL or AGPL code (Liblast the game, Spelloff, others): distributing a build would oblige releasing this project under the same license.
- Record any borrowed code or assets with their license in `CREDITS.md`.
- Verify a repo's license before using it; no license means no reuse.

## Art
Low-poly, flat-shaded, vertex-colored meshes generated in code (`SurfaceTool`, CSG for blockouts), shared resources or `MultiMeshInstance3D` for repeated props. CC0 packs (Kenney, Quaternius, Poly Haven) are a fallback. Keep one cohesive palette.

## Open questions (ask before deciding)
**Decided**
- Friends use Windows: Windows client build only.
- Baseline weapon is hitscan for the MVP; projectile weapons come later through the fire-mode abstraction (see Weapons).

**Decide before milestone 1 to 3**
- Pin a Godot version (latest stable 4.x).
- Netcode approach: full client-side prediction and reconciliation, or client-authoritative movement with server-authoritative combat and polarity (simpler for friends, weaker against cheating). Decide before milestone 3.

**Decide before milestone 2**
- Exact toggle cost model (cooldown only vs. windup vs. resource meter).
- What the absorb meter is used for.

**Later (backlog)**
- Weapon acquisition: pickups, per-polarity, or draft.
- Whether to add dash or bhop on top of slide.
- Game modes for 3 to 8 players (see Player count section) and whether to add a third polarity.
- Camo strength (how transparent) and whether map zones flip on timers or switches.
- Whether to add room codes (multiple simultaneous games) or stay with one shared lobby.
- Whether affinity can change between rounds, whether to use point budgets, and whether to add a second pre-match choice.
- Sniper details: charge time, laser/glint visibility, neutral variant, and when to add lag compensation.
- Final game name (working repo name: color-shooter; fun candidate: Chroma Blast 3000).
