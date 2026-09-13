# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Artemis II Mission Tracker — an interactive 3D simulator for the NASA Artemis II mission, the first crewed journey to lunar distance since Apollo 17. Launched April 1, 2026 from Kennedy Space Center, Orion carried four astronauts on a 9-day free-return trajectory around the Moon.

## Commands

```bash
npm install
npm run dev      # tsx watch src/server.ts — dev server with hot reload at http://localhost:3000
npm test         # vitest run — all tests
npm run build    # tsc — compile src/ to dist/ (tests/ excluded, see tsconfig.json)
npm start        # node dist/server.js — run the compiled build
```

Run a single test file or test case directly with vitest (there is no separate `test` binary wired into package.json beyond `vitest run`):

```bash
npx vitest run tests/orbital-math.test.ts
npx vitest run -t "name of the test"
```

There is no lint script configured.

## Architecture

**Stack**: Express (TypeScript, ESM, compiled with `tsc`) serving both a JSON API and the static frontend; no bundler — `public/` is served as-is, and `public/js/app.js` is a single plain ES module that imports Three.js and Chart.js straight from CDNs via an `importmap` in `public/index.html` (`three@0.160.0` from unpkg, `chart.js@4.4.1` from jsdelivr).

**Data flow — read this before touching trajectory or telemetry code**: the trajectory is generated once at server startup (`generateTrajectory(600)` in `src/routes/trajectory.ts` and independently again in `src/routes/telemetry.ts` — each route module calls `generateTrajectory` itself, so there are two in-memory copies). The frontend (`public/js/app.js`, `init()`) fetches `/api/trajectory`, `/api/events`, and `/api/crew` **once** at load time, then drives the whole simulation client-side: its own MET clock, playback speed control, and position interpolation (`getPositionAtMET`, `formatMET` — reimplemented in JS, not shared with the TS backend). The `/api/telemetry/current` and `/api/telemetry/:met` endpoints implement a *second*, independent MET simulation in `src/routes/telemetry.ts` (module-level `simStartTime`/`simMET`/`simSpeed`, advanced via `Date.now()`) — this state is **not** currently consumed by the frontend and resets whenever the server restarts. Keep this duplication in mind: a change to playback/interpolation logic generally needs to be made in both `src/lib/orbital-math.ts` (backend) and `public/js/app.js` (frontend), and a change to simulated "current time" behavior only affects `/api/telemetry/*`, not what the 3D view shows.

**Trajectory generation** (`src/lib/trajectory.ts`): a hardcoded `WAYPOINTS` array of `[met_seconds, x, y, z]` control points is interpolated with Catmull-Rom splines (`catmullRom` in `src/lib/orbital-math.ts`) into `numPoints` samples; velocity is then derived per-point via finite differences over neighboring samples. It is deliberately not orbital-mechanically accurate — see `Contributing Real NASA Data` below for how to swap in real ephemeris data.

**Mission phase strings** (`getMissionPhase` in `src/lib/orbital-math.ts`) and all mission event content (`src/data/events.ts`) are written in **Spanish**, while code, comments, and this doc are in English. When adding events or phases, match the existing Spanish content, not the surrounding English code comments.

**MET representation**: seconds as a number internally, formatted to `DDD:HH:MM:SS` via `formatMET`/parsed via `parseMET` (`src/lib/orbital-math.ts`) only at the boundary (API JSON, `src/data/events.ts` source literals, UI display).

**Verifying changes**: the user does not want Playwright (or other browser automation) used to check the app. Verify via `npm test`, the API responses (`curl`), and server logs instead.

**Claude skills** (`.claude/skills/`): `add-event` and `capture-view` automate common edits (`src/data/events.ts`, `public/js/app.js`) — if those files move or their shapes change, update the skills too, they drift silently otherwise. `scene-conventions` documents Three.js scale/color conventions for the 3D scene specifically; its palette overlaps but isn't identical to the HUD colors in `UI Color Conventions` below, so check both before changing a color.

## 3D Coordinate System

- **Units**: 1 unit = 1,000 km
- **Origin**: Earth center (0, 0, 0)
- **Y-axis**: Up (ecliptic north)
- **Moon position at T+0**: Along +X axis at (384.4, 0, 0)
- **Moon orbit**: Visual only — orbits Earth in ~27.3 days
- **Orion scale**: Exaggerated ~500× real size for visibility
- **Star field**: Spherical shell at r=800–1000 units

### Scale reference
| Object | Real size | Rendered size |
|--------|-----------|---------------|
| Earth | 12,742 km | 12.742 units (radius 6.371) |
| Moon | 3,475 km | 3.475 units (radius 1.737) |
| Earth-Moon distance | 384,400 km | 384.4 units |
| Orion capsule | ~5 m diameter | ~0.8 units (500× exaggerated) |

## Mission Data

### Key Events
| MET | Event |
|-----|-------|
| 000:00:00:00 | Launch from KSC |
| 000:01:57:00 | Trans-Lunar Injection burn |
| 004:18:37:00 | Enter Moon's sphere of influence |
| 005:07:56:00 | Distance record (surpasses Apollo 13's 400,171 km) |
| 005:18:44:00 | Loss of Signal (behind Moon) |
| 005:19:02:00 | Closest approach (~6,545 km above surface) |
| 005:19:07:00 | Maximum distance from Earth (406,732 km) |
| 005:19:25:00 | Acquisition of Signal + Earthrise |
| 009:07:10:00 | Splashdown, Pacific Ocean |

### Trajectory
The trajectory is a simplified free-return figure-8 path generated with Catmull-Rom splines through 23 control waypoints. It is not orbital-mechanically precise but is visually convincing. The real trajectory data can be substituted when NASA publishes it.

## API Reference

| Endpoint | Description |
|----------|-------------|
| `GET /api/trajectory` | All 600 trajectory points with telemetry |
| `GET /api/events` | All 25 mission events |
| `GET /api/events/upcoming?met=N` | Next 3 events from given MET |
| `GET /api/telemetry/current` | Current simulated telemetry (server-side simulation, unused by the frontend — see Architecture) |
| `GET /api/telemetry/:met` | Telemetry at specific MET (seconds) |
| `GET /api/crew` | Crew member information |

## UI Color Conventions

| Color | Use |
|-------|-----|
| `#00ff88` | Primary HUD text, active elements |
| `#1e3a5f` | Panel borders, inactive elements |
| `#6b8fa8` | Dim labels |
| `#ff8c00` | Current trajectory segment, warnings |
| `#0a0a0f` | Background, deep space |

## Adding Events

Add new events in `src/data/events.ts`:

```typescript
{
  id: 'unique-id',
  title: 'Event Title',
  met: '005:19:07:00',   // DDD:HH:MM:SS
  description: 'What happened and why it matters.',
  phase: 'LUNAR FLYBY',
  icon: '📡',
}
```

## Contributing Real NASA Data

When NASA publishes the actual Artemis II trajectory data:

1. Replace the `WAYPOINTS` array in `src/lib/trajectory.ts` with real ephemeris data
2. Update event METs in `src/data/events.ts` with actual mission times
3. Add real texture URLs to `public/js/app.js` (NASA Visible Earth: https://visibleearth.nasa.gov/)
4. The coordinate system uses km units — convert from km to units by dividing by 1000

### NASA Texture Sources (when available)
- Earth: NASA Blue Marble (2048×1024 JPEG)
- Moon: LRO WAC Global Mosaic

## Crew

| Name | Role | Agency |
|------|------|--------|
| Reid Wiseman | Commander | NASA |
| Victor Glover | Pilot | NASA |
| Christina Koch | Mission Specialist 1 | NASA |
| Jeremy Hansen | Mission Specialist 2 | CSA |

Jeremy Hansen is the first Canadian to travel beyond low Earth orbit.
