---
name: add-event
description: Add a new mission event to the Artemis II timeline
disable-model-invocation: true
argument-hint: "<title> <MET-timestamp> <description>"
allowed-tools: Read Write Edit Bash(npm test *)
---

# Add Mission Event

Add a new event to the Artemis II mission timeline.

## Arguments
- $0 = Event title (e.g., "Lunar Flyby Closest Approach")
- $1 = Mission Elapsed Time timestamp (e.g., "005:19:02:00" for day 5, 19h, 2m)
- $2 = Description

## Steps
1. Read `src/data/events.ts` to see the current event format (`id`, `title`, `met`, `description`, `phase`, `icon`)
2. Add the new event in chronological order within the `rawEvents` array
3. Assign a new unique `id` (kebab-case, following the existing entries)
4. Determine the `phase` from the MET by converting $1 with `parseMET` and passing the result to `getMissionPhase` (both in `src/lib/orbital-math.ts`) — use its returned Spanish string verbatim (e.g. `'LANZAMIENTO'`, `'VUELO DE IDA'`, `'SOBREVUELO LUNAR'`); do not invent a new phase name
5. Write the `description` in Spanish, matching the tone of existing entries
6. Run `npm test` to verify nothing broke
7. Report the event added with its `id` and `phase`
