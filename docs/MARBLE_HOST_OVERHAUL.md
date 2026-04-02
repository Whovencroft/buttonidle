# Marble Host Overhaul

## Goal
Turn the current button idle prototype into a small game host that can:
- keep the existing button idle game intact
- swap between multiple playable scenes
- support a real-time marble minigame without forcing that logic into the current DOM-heavy loop
- leave room for future board games, extra idle games, and larger subgames

## Why this overhaul comes first
The current project is still one large DOM-driven script. That works for the button idle loop, but a marble game needs:
- its own simulation update loop
- its own renderer
- direct input control
- collision and stage data
- pause, fail, restart, and reward handoff

If marble logic is added directly into the current monolithic `main.js`, the host will become harder to extend every time a new game mode is added.

## End State
The app should become:
- a persistent shell
- a scene host inside the Play tab
- one scene module for the existing button idle game
- one scene module for the marble minigame
- shared save data with per-scene save slices

## Target structure

```text
index.html
css/
  styles.css
  scenes/
    marble.css
js/
  main.js
  core/
    app_state.js
    save.js
    scene_manager.js
    input_router.js
  scenes/
    button_idle_scene.js
    marble_scene.js
    marble/
      marble_state.js
      marble_renderer.js
      marble_physics.js
      marble_levels.js
      marble_input.js
```

## Required architectural steps

### Step 1: Add a scene host layer
The Play panel should stop assuming the button game is always active.

Create a single `sceneHost` mount point inside the Play area. The app shell remains DOM-based.

The scene host must support:
- mounting a DOM scene
- mounting a canvas scene
- clean scene enter/exit hooks
- scene-local update and cleanup

### Step 2: Add top-level scene state
The save schema needs a neutral scene entry point.

Add:

```js
state.app = {
  activeScene: 'button_idle'
};

state.scenes = {
  button_idle: {},
  marble: {}
};
```

The button idle scene will eventually move from the current top-level shape into `state.scenes.button_idle`.

### Step 3: Create a scene manager
The app needs a small manager that can:
- register scene modules by id
- activate one scene at a time
- call `enter()`, `exit()`, `update(dt)`, and `render()` as needed
- keep shell UI and scene UI separate

### Step 4: Extract the current button game into a scene module
Before marble exists, the current idle loop should become `button_idle_scene`.

This means moving button-specific logic out of the shell path:
- manual press
- fake buttons
- popups
- autonomy ending
- button render/update functions

The goal is not to rewrite all button logic immediately. The first pass is to wrap and isolate it.

### Step 5: Add a canvas-backed marble scene
The marble game should use a dedicated canvas inside the scene host.

The first version should be intentionally small:
- top-down or pseudo-isometric presentation
- one marble
- one stage
- walls
- holes or fail zones
- finish goal
- timer
- reset and restart

### Step 6: Separate simulation from rendering
For the marble scene, keep these concerns separate:
- state
- update/physics
- input
- rendering
- level data

Do not bind movement rules directly to drawing code.

### Step 7: Define reward handoff back to the shell
The marble scene should return structured results, not direct shell mutations.

Example:

```js
{
  completed: true,
  bestTimeMs: 38210,
  reward: {
    presses: 5000,
    unlocks: ['marble_shop']
  }
}
```

The shell then applies the reward.

## Marble minigame scope for v1
Keep the first playable version narrow.

### In scope
- arrow or WASD movement
- touch overlay later
- acceleration and deceleration
- basic friction
- circle vs rectangle collision
- static walls
- fail pits
- end goal
- one small stage
- pause and restart
- reward callback to the shell

### Out of scope for v1
- full 3D
- rotating camera
- moving enemies
- complex slopes
- level editor
- multiple hazard families
- procedural stages
- mobile polish

## Physics choice for v1
Use simple custom 2D physics.

Do not begin with engine-grade physics or fake 3D. The first goal is feel and structure.

Recommended first-pass model:
- position `x, y`
- velocity `vx, vy`
- input acceleration
- drag/friction
- capped speed
- collision resolution against stage rectangles
- circle hitbox for the marble

This is enough for a convincing prototype.

## Milestones

### Milestone A: Host refactor
- add scene host DOM mount
- add app-level scene state
- add scene manager
- wrap current button game as `button_idle_scene`
- no visual change required beyond structure

### Milestone B: Marble shell scene
- mount a canvas scene
- show placeholder level bounds
- show a controllable marble
- no hazards yet

### Milestone C: Playable prototype
- collisions
- fail zone
- goal zone
- timer
- restart
- reward handoff

### Milestone D: Content pass
- 3 to 5 stages
- stage select or unlock path
- shell integration rewards

## First code changes to make next
1. Modify `index.html` to add a neutral `sceneHost` in the Play panel.
2. Add app-level `activeScene` and per-scene save slices.
3. Create a minimal `scene_manager.js`.
4. Move current button logic behind a `button_idle_scene` wrapper.
5. Add `marble_scene.js` with a canvas and placeholder update loop.

## Important constraint
Do not try to fully modularize every existing button function in one pass. First create boundaries. Then move logic across those boundaries in controlled chunks.
