---
name: blender-step-build
description: Build or animate a Blender scene live, step by step, through the omnix MCP so each step visibly appears in the viewport at a pace suited to screen-recorded demos. Use when the user wants to show Claude + Blender working, a "watch it build" demo, a time-lapse style build-up, objects dropping or sliding into place, or a turntable animation. Not for one-shot scene generation where nobody is watching.
---

# Blender step-by-step build (live demo pacing)

Goal: make a scene appear in the user's open Blender in small, visible steps, instead of all at once, so it works as video footage of "Claude + Blender".

## Prerequisites (check first)

1. `blender.scene_info` works (Blender running, omnix add-on started: N panel > Omnix > Start).
2. `blender.run_python` is enabled. It is off by default; the user must tick "Allow run_python" in the add-on preferences. Test with `result = 1+1`. Without it you are limited to the structured tools (`create_primitive`, `set_transform`, `add_modifier`, ...), which also work but are less flexible.
3. Read `blender.blender_guide` once: metres, Z-up, rotation tools take DEGREES, Python API uses radians.

## Core rules

- **One visible change per call.** Blender only redraws between requests, so a single big `run_python` shows nothing until it finishes. Do each step in its own `blender.run_python` call.
- **Pace with a busy-wait, not a timer.** The omnix `execute` sandbox has no `setTimeout`. Use:
  ```js
  const wait = ms => { const t = Date.now(); while (Date.now() - t < ms) {} };
  ```
  This blocks only the sandbox, not Blender, so the viewport keeps redrawing during the wait. Never use `time.sleep` inside `run_python`: it freezes Blender's UI and hides the very thing you are showing.
- **Typical pacing:** 1-2.5 s after a major step, 0.6-1.2 s between waves of small items, 60-100 ms between frames of a motion. Each call has its own round-trip, so real gaps are slightly longer than the numbers. Ask the user if the feel is off and adjust.
- **Keep one `execute` under a few minutes.** Split long builds into two or three `execute` calls (for example: setup + first objects, then the rest + camera) to avoid timeouts.
- **Shared helpers must be re-declared in every `run_python` call.** Python state does not persist between calls. Keep helpers in a JS string (`const DEFS = ...`) and prepend it: `py(DEFS + "...")`. The JS `state` object persists between `execute` runs, the Python side does not.
- **Never render unless asked.** `render_image` blocks the UI. If the user says "don't render", only build and save.
- **Never overwrite the user's existing .blend.** Start a new file for the demo and save under a new name.

## Recommended flow

1. **Preflight.** `scene_info`. If a previous animation is playing, run `bpy.ops.screen.animation_cancel(restore_frame=True)`. A playing animation also marks the file "unsaved", so `unsaved: true` can be a false alarm; if in doubt, save the old file first.
2. **Fresh scene.** `blender.new_file({discard_changes: true})` (only after the old file is saved), delete default objects, set the world colour, set the engine and resolution, set the 3D viewport to `RENDERED` or `MATERIAL` shading:
   ```python
   for w in bpy.context.window_manager.windows:
       for a in w.screen.areas:
           if a.type == 'VIEW_3D':
               a.spaces[0].shading.type = 'RENDERED'
   ```
3. **Frame the viewport** before the first object so the audience sees it appear: set `region_3d.view_location`, `view_distance`, `view_rotation` (an Euler converted to quaternion). Switch to `view_perspective = 'CAMERA'` only at the end, after the camera exists. That switch is a good reveal moment.
4. **Build in layers**, in this order of visual payoff: bare shape, then shading/material, then environment (floor, lights), then details, then secondary objects, then composition (camera).
5. **Details in waves.** Add many small items (scatter, particles, decals, props) in several batches, one colour or group per batch, with a short wait between. Looks much more alive than one batch.
6. **Motion by interpolation.** Move things in 6-14 small steps with smoothstep easing, each step a tiny `run_python`:
   - ease in/out: `e = t*t*(3-2*t)`
   - drop with deceleration: `e = 1 - (1-t)**2`, `z = target + height*(1-e)`
   - Spawn the object offset from its target, then step it into place. Group multi-part objects under an Empty and animate the Empty.
7. **Final animation (optional).** For a turntable, parent everything except floor/camera/lights to a rig Empty and keyframe its Z rotation 0 to 360 deg over N frames. Set linear interpolation *before* inserting keys, because the fcurve API changed in Blender 4.4+/5.x and editing fcurves directly is fragile:
   ```python
   bpy.context.preferences.edit.keyframe_new_interpolation_type = 'LINEAR'
   rig.rotation_euler = (0,0,0);                 rig.keyframe_insert('rotation_euler', index=2, frame=1)
   rig.rotation_euler = (0,0,math.radians(360)); rig.keyframe_insert('rotation_euler', index=2, frame=N+1)
   scene.frame_end = N   # last frame equals first, so the loop is seamless
   ```
   Start playback with `bpy.ops.screen.animation_play()`; stop with `animation_cancel(restore_frame=True)` before saving.
8. **Save last**, under a new filename, via `bpy.ops.wm.save_as_mainfile(filepath=...)`. Blender then leaves a `.blend1` backup next to it on later saves; delete it if the user does not want it.

## Parenting gotcha

When parenting existing objects to a new Empty that is not at the origin, set `child.matrix_parent_inverse = parent.matrix_world.inverted()` so the child does not jump.

## Honesty about what you can see

`render_image` and `viewport_screenshot` return image handles you may not be able to open. If you cannot actually view the result, say so plainly and ask the user to check the viewport; do not describe the look as verified. Report what was executed and what was not seen.

## What to tell the user at the end

- The ordered list of steps that appeared.
- Where the file was saved (and that nothing was rendered if so).
- Which numbers control pacing so they can ask for faster/slower.
- Offer: replay the build in a fresh file for a clean recording take.
