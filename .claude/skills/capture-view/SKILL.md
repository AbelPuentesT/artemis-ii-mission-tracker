---
name: capture-view
description: >
  Generate a specific camera view of the Artemis II mission at
  a given time. Use when the user asks to see a particular
  moment of the mission.
disable-model-invocation: true
argument-hint: "<MET-time> <view-preset>"
---

# Capture Mission View

Set up a specific camera view at a mission time point.

## Arguments
- $0 = MET time (e.g., "005:19:00:00" for lunar flyby)
- $1 = View preset: "earth", "moon", "follow", "overview", "earthrise" (the exact modes `setCameraPreset` accepts in `public/js/app.js`)

## Steps
1. Read `public/js/app.js` and use `getPositionAtMET` (~line 465) to find the Orion spacecraft position at MET $0 — this is the frontend's own trajectory interpolation, independent of the backend copies in `src/lib/trajectory.ts`
2. For the Moon's position at that MET, mirror the logic in `moonPositionAtMET` (`src/lib/orbital-math.ts`) — it isn't currently duplicated in the frontend, so compute it inline or add the equivalent helper to `app.js`. Earth is always at the origin (0, 0, 0), so it needs no lookup
3. Calculate the ideal camera position for view "$1", following the existing presets in `setCameraPreset` (`public/js/app.js`):
   - **earth**: Camera behind Orion looking back at Earth
   - **moon**: Camera near Moon surface looking at approaching Orion
   - **follow**: Camera trailing Orion with Moon/Earth in background
   - **overview**: Zoomed out showing full trajectory
   - **earthrise**: Camera angle replicating the famous Earthrise photo
4. `setCameraPreset`'s existing presets are static (not MET-aware, aside from `follow` which tracks the live `orionRef`); to bookmark a *specific* MET, add a new branch (or extend `animateCameraTo`) that uses the MET-specific Orion/Moon positions from steps 1–2 instead of their current live positions
5. Report the camera coordinates and what the user will see
