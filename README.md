# ECHO

A browser puzzle platformer where a ghost copy of you replays your movements from two seconds ago, and you use that copy to get through each room. The whole game is one HTML file with plain JavaScript, the Canvas 2D API and the Web Audio API. It uses no libraries, image files or sound files.

Built at TartanHacks 2026 by Alan Dong, Bryan Yung, Seunghyun Bae and Michael Li.

![ECHO cover art](echo_thumbnail.png)

## How it works

Everything lives in `index.html`: the CSS and HTML overlays at the top, and one `<script>` block that holds the game.

1. **Game loop.** `update()` runs on `requestAnimationFrame` but only advances when 1/30 of a second has passed, so the game ticks at a fixed 30 frames per second. Locking the rate matters because the ghost delay is counted in frames, not milliseconds. The loop switches on `gameState`: the animated title screen, a short text cutscene, then `PLAYING`, which calls `renderLevel()` every tick.

2. **Input.** Arrow keys move and jump, `1`/`2`/`3` pick a ghost mode, and `S` teleports. On phones, an on-screen joystick and jump button (built with pointer events) write into a `virtualInput` object that the movement code reads alongside the keyboard state, so touch and keyboard share one movement path. The mode and teleport buttons call the same functions as the `1`/`2`/`3` and `S` keys.

3. **Player physics.** Each tick adds input to velocity, applies friction (`vx *= 0.85`) and gravity (`vy += 1.2`), then moves the player one axis at a time. After each axis move, `resolve()` checks the player box against every solid tile's hitbox and pushes it back out. Moving one axis at a time means the code always knows which side of a tile to push the player out of, and landing on top of a tile is what sets `grounded`. A short coyote-time window (6 frames) still allows a jump just after walking off a ledge.

4. **The ghost.** Every tick the player's position is pushed onto `p.history`. Once the buffer holds more than 60 entries (2 seconds at 30 fps), the oldest entry is shifted off and becomes the ghost's position. That is the whole replay system: a fixed-length queue of past positions. The HUD's "TETHER" percentage shows how full that buffer is, and `drawGhostTrajectoryWorld()` draws the buffered path with moving arrows so you can see where the ghost is about to go.

5. **Ghost modes.** The ghost has three modes, defined in the `MODES` array. Only BOUNCE is available at the start; the other two become available when the player touches a module tile (`collectUpgradeModules()`).
   - **BOUNCE:** the ghost carries a spring on its head. `resolve()` checks if the falling player lands on it and launches them upward, a little higher than a normal jump.
   - **SUPER-LIGHT:** the ghost becomes a moving light source in the lighting pass. This matters in BLACKOUT LAB, where the player's own light radius is very small.
   - **ANCHOR:** `teleportToGhost()` moves the player to where the ghost currently stands. The final level's ending is built on this: when a cage traps the player, teleporting to the ghost swaps the two and leaves the copy behind.

6. **Levels.** Each level in the `levels` array is a 2D grid of tile IDs plus a spawn point, dialogue triggers and optional lighting settings. The `TILES` table maps each ID to a hitbox, a solid flag and a `draw()` function, so adding a tile type or a level is mostly data. There are seven rooms, from CELL to THE DOOR. Touching a vent tile calls `transitionLevel()`, which loads the next grid and clears the ghost buffer. Laser tiles call `respawnAtLevelStart()` on contact, and the ANCHOR CHASM vent only opens for a few seconds after you hit a remote switch.

7. **Dialogue.** `checkTriggers()` looks up the player's current tile (or column) in the level's `triggers` map and shows a line once. Short tips and respawn quips float above the player's head through `showHeadCallout()`.

8. **Rendering and lighting.** `renderLevel()` draws in layers: background, tile grid, ghost path, ghost, player, then particles. Lighting uses a second off-screen canvas (`lightBuffer`). It is filled with a dark layer, and then light shapes (the player's radius, flickering lamp cones, the SUPER-LIGHT ghost, glowing modules) are cut out of it with `destination-out` compositing. The buffer is drawn over the scene, and a final pass with `lighter` blending adds glow so light sources look bright instead of just less dark.

9. **Audio.** All sound is generated at runtime. `playSound()` builds short effects from oscillators with pitch and volume ramps (jump, respawn, level change), and footsteps are filtered white noise. Background music comes from `scheduleAmbientCycle()`, which schedules pad, bass and melody notes from a small chord progression and re-queues itself before the loop ends. The audio context is only created on the first key press, click or tap, because browsers block audio until the user interacts with the page.

## Project layout

```
index.html        the full game (markup, styles and script)
serve.sh          unpacks a .tar.gz/.tar.br/.tar.zst archive into extracted/ and serves it on port 8000
game.tar.br       brotli-compressed tarball of a minified build from an earlier commit
extracted/        the unpacked minified build (index.html)
echo_thumbnail.png  cover art
GAME.md           longer design notes written during the hackathon (some details predate the final level set)
docs/plans/       design notes for the level system and a multiplayer idea that was not merged
```

## Tech stack

- JavaScript (no frameworks or build step)
- HTML5 Canvas 2D API
- Web Audio API
- Pointer events for touch controls
- Bash and Python's built-in HTTP server for local serving

## Running it

The game is a single static file. Open `index.html` in a browser, or serve the folder locally:

```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

To run the packaged minified build instead (needs the `brotli` command-line tool):

```bash
./serve.sh game.tar.br
# then visit http://localhost:8000
```

Controls: arrow keys to move and jump, `1`/`2`/`3` to switch between the ghost modes you have collected, `S` to teleport to the ghost in ANCHOR mode. On a phone, use the on-screen joystick and buttons.
