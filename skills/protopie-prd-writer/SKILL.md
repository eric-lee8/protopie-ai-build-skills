---
name: protopie-prd-writer
description: Write a PRD that an AI agent (Claude Code with the ProtoPie Studio MCP) can use to build a ProtoPie prototype's interactions in one shot from a design imported from Figma. Covers the PRD structure, how to prepare Figma frames so they import cleanly, how to specify signals, states, animations and acceptance tests, and the ProtoPie and MCP limits the spec has to plan around. Use this skill whenever someone wants to write, review or improve a PRD, spec or requirements doc for a ProtoPie prototype, plan an AI-built ProtoPie interaction, prepare Figma designs for a ProtoPie MCP build, or connect hardware (ProtoPie Connect, steering wheels, sensors, controllers) to a prototype, even if they just say "spec this prototype".
---

# Writing PRDs for AI-built ProtoPie prototypes

A PRD for an AI build is not a normal product spec. The agent reads it literally, can only use what the ProtoPie Studio MCP supports, and fills any gap with a guess. A good PRD removes the guesses: exact layer names, exact numbers, intent for every feature, and tests that prove it works.

This skill comes from a real build: an automotive instrument cluster driven by a Logitech G29 wheel through ProtoPie Connect. The finished build had 44 triggers, `inspect_interactions` reported 0 issues, and it took about 15 minutes from one prompt. Examples below use that project; the rules apply to any ProtoPie prototype.

Pair this skill with **protopie-build-prompt**, which covers the prompt that hands the PRD to the agent.

---

## 1. The core split: designer owns visuals, AI owns interactions

State this at the top of every PRD, and design the whole document around it.

- **Designer:** builds the screens in Figma, prepares the frames (section 3), imports them with the ProtoPie Figma plugin, and does the few manual steps the MCP can't.
- **AI:** checks layer names, sets starting states, makes a short list of *allowed preparation edits*, then builds variables, triggers, conditions and responses.
- **The AI never draws, redraws or restyles layers.** If a layer is missing, it stops and asks. Every visual the interactions need must already exist in the import.

Why it matters: an agent that can draw will "fix" a missing layer with a substitute that looks wrong, and you won't notice until the demo.

---

## 2. PRD structure

Use these sections, in this order. Numbered sections let the prompt say "PRD section 6 is binding".

| § | Section | What it must contain |
|---|---|---|
| 1 | **Overview** | Goal, the designer/AI build workflow, success criteria (measurable, e.g. "`inspect_interactions` reports 0 issues") |
| 2 | **Scope** | In scope / out of scope columns, plus the **one-mechanic rule** (section 4.1 below) |
| 2.1 | **Intended behaviour** | A moment-by-moment table: *Moment · User does · Prototype should · Why it matters*, plus 4–5 behaviour principles. This is also the demo script. The prompt tells the AI to resolve ambiguity in favour of this section |
| 3 | **Architecture** | Scenes and what each is for (section 3.3), variable scope (GLOBAL), signal flow diagram |
| 4 | **Layer inventory** | Table: *Layer name · Type · Initial state · Driven by*. Then **allowed preparation edits** (section 3.4) |
| 5 | **Signal requirements** | Inputs → messages → variables → handling → visible result; pie-to-pie messages; derived values; normalization |
| 6 | **Interaction and animation requirements** | One block per feature: **Intent · Mechanic · Timing · Accept** |
| 7 | **Variables** | *Name · Initial · Source*, including helpers, plus "the AI may add helpers and must list them" |
| 8 | **Visual tokens** | Colors, fonts and their uses |
| 9 | **Prerequisites** | Import steps, a post-import spot-check, MCP and skill installed, optional permissions |
| 10 | **Manual setup** | Everything the MCP can't do, as numbered steps |
| 11 | **Acceptance tests** | Numbered tests (T0, T1 …) runnable in simulation or on hardware |
| 12 | **Assumptions** | What's unconfirmed, rig-specific values, expected build time |
| 13 | **Agent prompt** | The copy-paste prompt (write it with **protopie-build-prompt**) |

---

## 3. Prepare the Figma frames before writing the PRD

Most first-build failures came from the import, not the logic. Fix the design so it imports ready to wire.

### 3.1 Name every layer the PRD mentions, in Figma
The ProtoPie Figma plugin keeps layer names. If Figma says `Container` and the PRD says `Speed Value`, the agent has to guess the mapping. Rename in Figma first, then copy the exact names into the PRD inventory. Name supporting containers too (`Speed Readout`, `Gear D Slot`), so nothing is called "Container" or "Icon".

### 3.2 Give every animated part its own layer
- **Rotating parts** (pointers, dials, wheels): make each a separate layer whose bounds are centered on the pivot. A gauge pointer is a gauge-sized square with the needle drawn pointing up, anchored at the center. In the frame you import, draw it at **rotation 0**: importers may bake a rotation into the graphic, and then "set rotation to −135°" would stack on top of it.
- **Text that changes:** split value from unit and label (`118` / `km/h`, `STEER` / `−35°`). A mixed-style text node loses its styling when the agent rewrites it.
- **Moving highlights** (focus rings, underlines, "now playing" markers): make them separate layers positioned outside auto-layout flow, so they can move between items. Put the exact x/y for every position in the PRD.
- **Hidden helpers:** if a layer exists only to be revealed (`Icon Play` next to `Icon Pause`), create it in Figma at **opacity 0**. Don't use visibility off, because importers may skip hidden layers.

### 3.3 Decide what each frame is for
- **States of one screen** (driving / overspeed / mode overlay): build one **working frame** that contains every layer all states need, with the extra ones at opacity 0. The prototype runs in that one scene and produces each state with interactions. Other state frames are **optional references** for checking the result.
- **Separate screens** (Drive Modes / Music Player): import each as its own live scene. Shared parts (nav rail, climate bar) exist as copies in each scene, so the PRD must say "build their interactions in both".
- **Why one working frame:** the ProtoPie MCP **cannot move layers between scenes**. Its move call reports success but leaves the layer where it was. Anything a scene needs must arrive in that scene's import.
- Draw the working frame in its **idle state** where possible. Otherwise, list every starting value the agent must reset (section 3.4).

### 3.4 List the allowed preparation edits
The plugin imports many text layers as images and icons as vector images. The agent has to convert some of them, which technically touches visuals, so permit it explicitly:
- **Text:** convert every image-text layer an interaction changes to editable text, re-apply its weight (conversion drops bold), and set its idle value. List them by name. Static labels stay as images.
- **Icons:** set the starting color of every icon an interaction recolors, which converts it to a shape so `color` responses work. List them by name.
- **Mid-use values:** reset values the frame shows mid-use (bars at 68%, progress at 43%, a focus border on row 4). List each one with its starting value.

---

## 4. Writing requirements the agent can't misread

### 4.1 One basic mechanic per element
Every element is driven by exactly one of: rotate, move, scale, swap text, opacity, color/border, jump. Out of scope: animated scenes, arc-fill masks and particle effects. They take hours to tune, and the agent will try them if you don't forbid them. If a requirement seems to need more, specify the basic version and log the rest as a follow-up.

### 4.2 Every feature gets Intent, Mechanic, Timing, Accept
```
### F8 · Overspeed alert
- Intent: warn clearly without alarming; never flicker around the limit.
- Mechanic: Range trigger speed ≥ 120 → Speed Value color #FF5A5F, Nav Card opacity 0,
  Overspeed Banner opacity 100 + scale 95→100% from center.
- Timing: 0.25 s, ease out. Exit at speed ≤ 117 (3 km/h gap prevents flicker).
- Accept: hold full throttle → banner after ~4 s; brake below 117 → nav card returns.
```
The **Intent** line is what lets the agent make good choices when the spec is silent. The **Accept** line becomes a test.

### 4.3 Give exact numbers, never "full" or "matching"
- **Geometry:** "fill width = value% × 754, knob x = fill width − 14", not "the fill follows the knob".
- **Every position of a moving element:** "focus ring x: Eco 6, Comfort 161.5, Sport 317, Snow 472.5".
- **Every color per state**, including **glows and shadows**: a badge recolored green kept its orange glow because only the fill was specified. Also specify label contrast on light backgrounds.
- **Durations in seconds**, plus the easing.

### 4.4 Separate "focus" from "current" in selection UIs
Browsing and selection need different visuals. In this project the blue ring with a white label means "you're browsing this", and the orange outline means "this is active". Without the distinction, the first build marked the current mode only on one option.

### 4.5 Thresholds need hysteresis
Alerts turn on and off at different values (on at ≥ 120, off at ≤ 117). Otherwise they flicker on noisy input.

### 4.6 State every fact once, then cross-check
The first build hit a contradiction: the inventory said "ACC Off" at start, while the variables, the demo script and the tests said "ACC 110". Before handing off, check that each element's **initial state** agrees across the inventory (§4), the variables (§7), the demo moments (§2.1) and test T1.

---

## 5. Specifying hardware and external signals

For prototypes fed by ProtoPie Connect, plugins or sensors.

1. **Confirm the signals from two sources.** The official API docs give names and ranges. A **live capture** from the Connect message log shows what this rig actually sends. When they disagree, **trust the rig**: the G29's steering read 0 at full *left*, the reverse of its docs.
2. **A short capture shows names, not ranges.** Briefly touched inputs show a sliver of their range. Don't treat observed values as limits.
3. **Write a signal table:** *Input · Message · Variable · Raw value → handling · Visible result*.
4. **Normalize on arrival.** Store the raw value (`steerRaw`), then assign a contract variable (`steerAngle = (steerRaw − 50) / 50 × 450`). Interactions bind only to normalized variables, so a reversed or rescaled input is a one-formula fix.
5. **Buttons send press and release** (1, then 0). Store the value and act only on 1. Otherwise every toggle undoes itself. Add a "hold for 2 s" test.
6. **Pulse signals race.** A dial that sends ±1 followed by 0 milliseconds later overwrites the stored value before a formula reads it. Branch with conditions (`> 0` → +1, `< 0` → −1); never read the pulse variable inside the formula.
7. **Messages arrive only on change.** Anything that should keep changing while an input is held (speed while the pedal is held) needs a ticker loop that recomputes on a timer.
8. **Pre-answer rig facts** (ranges, connected accessories) in the PRD and prompt, so the build doesn't stop to ask.

### Multi-display prototypes
- **Shared hardware state:** every pie receives the same hardware messages, so each pie computes shared state identically. No messages between pies are needed.
- **Touch input:** only touch on one display needs a pie-to-pie message. The MCP **cannot author Send responses** on the ProtoPie Studio channel, so list them under Manual setup (§10).
- **Dependent values:** if a derived value depends on state (speed depends on the gear), every pie that shows the value must track that state too, even without displaying it.

---

## 6. ProtoPie and MCP limits to design around

Put the relevant ones in the PRD where they apply. The agent then doesn't rediscover them, which cost about 4 minutes of fixes in the first build.

| Limit | Plan for it |
|---|---|
| Can't move layers between scenes (false success) | One working frame per screen, with all its state layers inside |
| A rotation mapping segment caps at 360° | Rotations beyond ±360° use multi-stop mappings (e.g. −450 → −150 → 150 → 450) |
| No `if()` in formulas | Write branching as conditions ("two condition branches on `gearPos`") |
| `color` responses replace gradients with solid colors | Specify a solid stand-in color per state |
| `capture_preview` renders only a scene's starting state | Verify states by reading values back; optional macOS Screen Recording permission for live screenshots |
| Image text loses its weight when converted | List which values need bold re-applied |
| Connect plays the saved `.pie` file, not the open document | Manual setup: "save, then reload in Connect" |
| Send on the Studio channel can't be authored | Manual setup steps |

---

## 7. Acceptance tests

- **Number them** (T0, T1 …) so the agent can report pass, fail or needs-hardware per test.
- **Start with an idle-state test** (T1) that matches the starting states exactly.
- **Include the tests a build is most likely to get wrong:**
  - **Direction:** turn left → the icon turns left.
  - **Release safety:** hold the button 2 s, release → toggles once.
  - **Preconditions:** speed only rises in D.
  - **Wrap-around:** last track → first track.
  - **Precedence:** a dial press confirms the mode instead of playing a song.
- **Mark dependent tests:** "not applicable until the other display is built".
- **Add a health test:** `inspect_interactions` reports 0 issues.

---

## 8. Hand-off checklist

Before sending the PRD to a build:

- [ ] Every layer in §4 exists in Figma with that exact name, in the frame being imported
- [ ] Animated parts are separate layers; rotating parts are drawn at rotation 0 in the import frame
- [ ] Hidden layers are at opacity 0, not visibility off
- [ ] The working frame contains every state's layers; reference frames are marked optional
- [ ] Allowed preparation edits list the text, icons and starting values by name
- [ ] Every feature has Intent, Mechanic, Timing, Accept, with exact numbers and colors (glows included)
- [ ] Initial states agree across §2.1, §4, §7 and T1
- [ ] Signals are confirmed from the docs and a live capture, with press/release, pulses and normalization handled
- [ ] Rig facts are pre-answered; manual steps (Send responses, save + reload) are listed
- [ ] The tests cover direction, release, preconditions, wrap-around and health

After each build, add what the agent had to work around to the PRD (and to any project skill). The second build should never hit the first build's problems.
