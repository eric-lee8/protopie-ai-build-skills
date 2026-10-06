# ProtoPie AI Build Skills

Two agent skills for building **ProtoPie** prototypes with **Claude Code** and the **ProtoPie Studio MCP**. Bring your design in from Figma, write a spec, and let the agent build every interaction.

| Skill | What it helps you do |
|---|---|
| [`protopie-prd-writer`](skills/protopie-prd-writer/SKILL.md) | Write a PRD an AI agent can build from in one shot: structure, Figma frame preparation, signals, states, animations, acceptance tests, and the ProtoPie/MCP limits to plan around |
| [`protopie-build-prompt`](skills/protopie-build-prompt/SKILL.md) | Write the prompt that runs the build (scene setup, work order with a pause checkpoint, guard rails, report format), plus small fix prompts, session hygiene and live-demo planning |

## Where these come from

Both skills come from a real build. An automotive **instrument cluster**, driven live by a **Logitech G29** steering wheel through ProtoPie Connect, was built by Claude Code from one prompt and a PRD:

- About 15 minutes from prompt to saved pie
- 44 triggers, 0 issues in `inspect_interactions`
- Passed its hardware tests: steering, pedals, paddles, dial and buttons

Every rule in the skills exists because the build hit the problem:

- **Steering:** the wheel reported the opposite direction from its docs.
- **Buttons:** they fire on both press and release.
- **Dial:** its ±1 pulse races against a trailing 0.
- **Rotation:** a single rotation mapping is capped at 360°.
- **Layer moves:** moving layers between scenes falsely reports success.

## Install

### Claude Code: plugin (recommended)

From a terminal:

```bash
claude plugin marketplace add eric-lee8/protopie-ai-build-skills
claude plugin install protopie-ai-build@protopie-ai-build-skills
```

Newer Claude Code versions (2.1.275+) can also install from inside a session with `/plugin install protopie-ai-build --marketplace protopie-ai-build-skills`.

### Claude Code: personal skills folder

```bash
git clone https://github.com/eric-lee8/protopie-ai-build-skills.git
cp -R protopie-ai-build-skills/skills/* ~/.claude/skills/
```

The skills load in every session. To use them in one project only, copy them to `<project>/.claude/skills/` instead.

### Claude app

Download `protopie-prd-writer.skill` and `protopie-build-prompt.skill` from the [latest release](../../releases/latest). Open each one in the Claude app and choose **Save skill**.

## Use

Just ask. The skills trigger on requests like:

- *"Write a PRD for this Figma prototype so Claude Code can build it in ProtoPie."*
- *"Review my PRD before I hand it to the ProtoPie MCP."*
- *"Write the build prompt for this PRD."*
- *"Write a fix prompt: the badge glow should match the mode color."*
- *"How should I run this build live in a webinar?"*

The typical flow:

1. **Prepare Figma:** name the layers and set up the import frames (PRD skill §3).
2. **Write the PRD** with `protopie-prd-writer`.
3. **Write the prompt** with `protopie-build-prompt`, and put it in the PRD.
4. **Import and build:** import the frames with the ProtoPie Figma plugin, open a fresh Claude Code session and send the prompt.
5. **Review:** check the report, run the acceptance tests, and feed what you learn back into the PRD.

## Requirements

- [ProtoPie Studio](https://www.protopie.io/) with its MCP server enabled and connected to Claude Code
- The ProtoPie Figma plugin, to import designs
- [Claude Code](https://code.claude.com/), or the Claude app for writing PRDs and prompts
- Optional: ProtoPie Connect, for hardware inputs and multi-display prototypes

The skills teach a method; they don't run tools themselves. ProtoPie MCP behaviour can change between releases, so if something here no longer matches, please open an issue.

## Contributing

Issues and pull requests are welcome, especially new "the build hit this" lessons from other prototypes. Keep each rule tied to a real failure and its fix.

## License

[MIT](LICENSE)
