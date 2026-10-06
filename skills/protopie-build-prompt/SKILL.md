---
name: protopie-build-prompt
description: Write the prompt that hands a PRD to Claude Code so it builds a ProtoPie prototype's interactions through the ProtoPie Studio MCP, plus short follow-up prompts for fixes and polish. Covers the prompt structure, scene setup, the work order with a pause checkpoint, guard rails, pre-answered questions, the report format, session hygiene, and how to review a finished build. Use this skill whenever someone wants to write, review or improve a prompt for an AI-built ProtoPie prototype, run a ProtoPie MCP build, write a fix prompt for an existing pie, or plan a live AI build for a demo or webinar, even if they just say "write the prompt for this".
---

# Writing prompts for AI-built ProtoPie prototypes

The prompt is the contract between the designer and the agent. The PRD says *what* to build; the prompt says *how to run the build*: where to look, what order to work in, when to stop, what never to do, and what to report.

This skill comes from a real build: an automotive instrument cluster driven by a Logitech G29 wheel. The build took one prompt and about 15 minutes, finished with 44 triggers and 0 issues, and passed its hardware tests after a one-formula fix. Pair this skill with **protopie-prd-writer**; the prompt assumes a PRD with numbered sections.

---

## 1. Prompt structure

Use these blocks, in this order:

1. **Job and boundary** (one or two lines): what to build, and "I have already imported the UI from Figma. Your job is the interactions, not the visuals."
2. **Sources of truth:**
   - The project skill, if any, for signals and recipes.
   - The PRD's URL, with "read the whole page before you start" and which sections are **binding**.
   - The tie-breaker: "when a detail is ambiguous, choose what best serves PRD §2.1 (intended behaviour) and tell me what you chose".
3. **Visual reference:** the Figma file URL, which section to use, and which sections to ignore (old versions, archives).
4. **How the scenes are set up:** see section 2. This is the block that most often prevents a wasted run.
5. **Current setup:** the open file and canvas size, which Studio tab to work in, asset paths (audio, images), and that the MCP is connected.
6. **Work in this order:** numbered steps with a **pause checkpoint** after discovery (section 3).
7. **Rules:** guard rails, known traps, pre-answered questions, manual steps (section 4).
8. **Report:** what was built, the manual steps, assumptions, and every acceptance test marked pass, fail or needs hardware.

---

## 2. Describe the scene setup explicitly

Agents don't infer how scenes relate. Say it, including what's optional and what must not happen.

**States of one screen** (one working scene plus optional references):
```
- The prototype runs entirely in ONE scene: "Cluster / Working Scene". It already contains
  Mode Overlay and Overspeed Banner at opacity 0. Produce every state (driving, braking,
  overspeed, mode dial) inside that scene with interactions, never by jumping to another scene.
- Reference scenes are OPTIONAL. If present, use them to check each state, and don't add
  interactions to them, jump to them or delete them. If absent, don't ask for them; verify
  against the PRD's values instead.
```

**Separate screens** (all live):
```
- The two scenes "Drive Modes" and "Music Player" are both live; the rail jumps between them.
- The rail and climate bar exist in both scenes. Build their interactions in BOTH.
- Shared state lives in GLOBAL variables; each scene refreshes from them with a Start trigger.
```

**Always add:**
```
- Don't move layers between scenes: layer_manage reorder into another scene reports success
  but leaves the layer where it was. If something is in the wrong scene, ask me.
```

---

## 3. Work order, with a pause checkpoint

```
1. Discover: studio_session_init, query_design_system, inspect_layout with outputTree.
   Confirm required layers exist (name them). Check every layer in the PRD inventory exists;
   rename only if the import changed a name. Make the allowed preparation edits from the PRD.
   List anything missing, then PAUSE and show me the check before continuing.
2. Variables: create every variable in PRD §7 as GLOBAL NUMBER with its initial value.
3. Signals: one Receive trigger per message (PRD §5), channel STUDIO, with enableValue +
   storeVarName. [Add the handling rules: raw → normalized, press-only, pulse branches.]
4. Derived values: tickers and formulas from PRD §5.3.
5. Bindings: features from PRD §6 with the exact mappings, colors and durations.
6. Verify: inspect_interactions must report 0 issues. Simulate [list the key inputs] and read
   the values back for each state. [Name any precondition, e.g. "shift into D before testing speed".]
7. Save as <Name>.pie in <folder>. If the file exists, ask before overwriting.
```

Why the pause works:
- **Catches missing layers before anything is built:** it's the cheapest moment to find import problems. In the first run it caught two layers that hadn't been pasted in, before anything was wired.
- **Makes a good live demo moment:** the agent shows its check and the presenter approves it on screen.
- **Remove it** only when the import is proven and the prompt must run unattended.

---

## 4. Rules block: guard rails, traps and pre-answers

### Guard rails
- "Do not create, redraw, restyle or reposition visual layers. You may only correct names the import changed, set anchors and the PRD's starting states, make the allowed preparation edits, and add variables (including helpers), triggers, conditions and responses. If a layer is missing, stop and ask."
- "Build interactions only in [the working scene / these scenes]."
- "Never edit [other pie]. If other pies are open, call list_contexts and bind_context to <Name>.pie first, and show me the context you're bound to." A new session connects to whichever Studio tab it touches first.

### Known traps (paste the ones that apply)
- **Buttons:** they send 1 on press and 0 on release; act only on 1.
- **Pulse inputs:** a dial sends ±1 then 0. Branch with conditions, and never read the pulse variable inside a formula.
- **Direction:** the hardware direction can differ from the docs (the steering wheel was reversed). Use the PRD's normalization, and never bind interactions to raw values.
- **Rotation range:** one rotation mapping segment caps at 360°; use multi-stop mappings beyond that.
- **Previews:** `capture_preview` renders only a scene's starting state. Read values back to verify other states.
- **Send responses:** the MCP can't author Send responses on the Studio channel. List them as manual steps with the exact trigger and value.

### Pre-answer every question you already know the answer to
In the first build, the agent paused to ask about the wheel range and shifter. Put the answers in the prompt:
```
The rig: wheel range 900 degrees, no shifter connected. Don't ask about these.
Audio files track01.mp3 … track07.mp3 are in /path/to/audio/.
```

### Report format
"Finish with:
- what was wired automatically (message → variable → visual)
- the manual steps
- assumptions and choices
- the helper variables you added
- acceptance tests T0 … Tn, each marked pass, fail or needs hardware"

---

## 5. Fix and polish prompts (after the first build)

Keep them small and verifiable. Template:
```
Polish <scene> in <Name>.pie. The PRD was updated, so re-read <sections> first.
1. <change>, with exact values (colors, positions, formulas).
2. <change> …
3. Exception to the no-reposition rule: <one named layer and the exact fix>.
Then verify:
- inspect_interactions (0 issues)
- simulate <inputs> and read back <named properties> (expect <values>)
- <a shortcut for slow checks, e.g. "move the progress near the end, or temporarily
  shorten the track, to test auto-advance, then restore it">
List any new helper variables, then save <Name>.pie.
```
Then reload the file in ProtoPie Connect before testing on hardware. Connect plays the saved file, so an unsaved fix looks like a failed fix.

Update the PRD in the same breath, so the next full build includes the fix.

---

## 6. Session hygiene

1. **Use a fresh session per build,** opened in the project folder where the ProtoPie MCP is registered. No `mcp add` needed.
2. **Build one display per session.** Build the cluster, then start a new session for the center display.
3. **Back up the working pie** (rename it) before a re-test, so the new build can't overwrite your fallback.
4. **Close other pies in Studio,** or make the target the active tab.
5. **Copy the prompt fresh from the PRD** every time. It changes after each build.
6. **Optional:**
   - Turn on `/fast` mode: most of the build time is the agent writing, not the tool calls.
   - Grant macOS Screen Recording permission (before the session starts) so the agent can screenshot live states.

## 7. Planning a live build (demos and webinars)

- **Budget about 15 minutes** for a full build, plus variance. In the first build, only about 1 minute of that was tool calls; the rest was the agent planning and writing.
- **Start it early and keep presenting.**
  - **While it runs:** walk through the PRD's intent table, the skill and the signal map. Tell the "hardware disagreed with the docs" stories.
  - **At the pause checkpoint:** approve it live.
  - **In Studio:** glance at the interaction panel filling up.
- **Have a safety net:** a pre-built pie in another tab, and a time-lapse recording of a full run.
- **Finish with a small live fix prompt** (1–2 minutes) to prove nothing was staged.

## 8. Reviewing a finished build

1. **Export the session** and build a timeline of where the time went.
2. **Read the report's assumptions:** each one is a gap in the PRD. Close it.
3. **Read its bug fixes:** each one is a trap for the PRD, the prompt or the project skill.
4. **Run the hardware tests,** direction and release first. Then run the intended-behaviour script once, end to end.
5. **Check the visuals** the agent couldn't see: colors per state, glows, alignment, contrast.
