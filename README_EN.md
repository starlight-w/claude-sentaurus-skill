# Sentaurus TCAD End-to-End Agent Skill

[中文 README](README.md) | English README

Current version: `v0.2.1`

This skill was created and organized with the help of **Claude Code**. Its purpose is simple: make agents less careless when working with Sentaurus TCAD.

It is built first for Claude Code's Skills mechanism. You can also use it as plain Markdown instructions or project knowledge in Claude.ai, OpenAI Codex, OpenCode, OpenClaw, or any agent environment that can read custom instructions.

It is not a Sentaurus installer, and it does not include Synopsys proprietary files. What it provides is a workflow and a set of guardrails so an agent can keep TCAD work traceable and reproducible.

The workflow is:

```text
Problem definition → literature and documentation search → official examples check → SWB project tree → SDE/SDevice/SVisual setup → gsub submission → monitoring → log/plt/tdr diagnosis → visualization and report → knowledge capture → next iteration
```

## Supported agent environments

| Environment | Recommended use | Notes |
|---|---|---|
| Claude Code | Import the `.skill` package, or place the files under `~/.claude/skills/sentaurus-tcad/` | Recommended path; references can be loaded as needed |
| Claude.ai / Claude Desktop | Add `SKILL.md` and the relevant `references/` files as project knowledge or attachments | Useful for planning, review, and deck generation; commands still run on your Sentaurus machine |
| OpenAI Codex / OpenCode / OpenClaw | Place the Markdown files in the corresponding agents/skills/instructions directory | These tools may not import `.skill` packages, but the workflow and guardrails still apply |
| Other LLM agents | Read `SKILL.md` first, then load `references/*.md` when details are needed | You will need to adapt tool calls, file access, and shell permissions |

For new users: **this is an operating manual for agents, not a Sentaurus installer.** You need a licensed, working Sentaurus environment before this skill can help.

## When to use this skill

Use this skill when you want an agent to help with:

- Creating or repairing Sentaurus Workbench projects.
- Writing SDE geometry, doping, contact, and mesh scripts.
- Writing SDevice Physics, Math, Solve, Plot, and Save sections.
- Running Id-Vg, Id-Vd, BV, HeavyIon, SEB, TID, ESD, or thermal simulations.
- Diagnosing convergence failures, abnormal leakage, breakdown location, or SEB criteria.
- Working on GaN HEMT, p-GaN HEMT, radiation effects, and reliability simulations.
- Keeping simulation results visible in the SWB GUI and documented with plots and reports.

## Core rules

The skill instructs the agent to follow these rules:

1. **Research before simulation**  
   Physics models, material parameters, traps, polarization, avalanche, HeavyIon, and thermal boundary conditions should be backed by official examples, literature, or verified experience.

2. **All simulations must be visible in SWB**  
   New projects should be created through `swbpy2`; new experiments should be added to the SWB tree. Avoid orphan decks outside the tree.

3. **Use traditional SWB projects by default**

   ```python
   deck = Deck(project_path, False)
   ```

4. **Submit through gsub**

   ```bash
   gsub -q local:default -e <node-number> <project-path>
   ```

5. **Do not bypass SWB with direct SDevice runs**

   ```bash
   sdevice pp*_des.cmd
   ```

   Direct SDevice execution bypasses SWB state management, making results invisible or hard to track in the GUI.

6. **Start a one-shot background monitor after submission**

   ```bash
   until grep -qE "Good Bye|FATAL|Step-size is too small" n<N>_des.log 2>/dev/null; do sleep 60; done
   tail -20 n<N>_des.log
   ```

7. **Inspect both `.plt` and `.tdr` results**  
   `.plt` files show curves; `.tdr` files show spatial distributions. Do not rely only on terminal output.

8. **Persist results**  
   Produce at least a PNG plot, Markdown table, or report, and update progress notes.

## Repository layout

```text
claude-sentaurus-skill/
├── SKILL.md                         # Main skill entry point
├── references/
│   ├── new-device-preflight.md        # First-run environment checks on a new machine
│   ├── swbpy2-gsub.md                # SWB, swbpy2, gsub, GUI visibility
│   ├── sde-mesh-patterns.md          # SDE geometry, Boolean, contacts, doping, mesh
│   ├── sdevice-patterns.md           # SDevice Physics/Math/Solve/Plot/Save
│   ├── gan-hemt-and-seb.md           # GaN HEMT, BV, HeavyIon, SEB methodology
│   └── results-reporting.md          # SVisual, plt/tdr diagnosis, plotting, reports
├── evals/
│   └── evals.json                    # Example trigger prompts
├── dist/
│   └── sentaurus-tcad.skill          # Importable skill package
├── README.md                         # Chinese README
├── README_EN.md                      # English README
├── SECURITY.md                       # Security notes
└── LICENSE                           # MIT License
```

## Changelog

### v0.2.1

- Added simulation queue tracking guidance to `SKILL.md` and `references/swbpy2-gsub.md`.
- Clarified that the queue script is only an auxiliary log and does not replace SWB/gsub state management.
- Recommended workspace script path: `scripts/sentaurus/sim_queue.py`; recommended state file: `claude_tmp/sentaurus/sim_queue.json`.

## Installation

### Option A: Import the `.skill` package

If your agent environment supports `.skill` packages, import:

```text
dist/sentaurus-tcad.skill
```

In Claude Code, after importing, you can ask: "Use the sentaurus-tcad skill to ...". If another agent platform does not support `.skill` files, use Option C.

### Option B: Manual installation for Claude Code

```bash
mkdir -p ~/.claude/skills/sentaurus-tcad
cp SKILL.md ~/.claude/skills/sentaurus-tcad/
cp -r references ~/.claude/skills/sentaurus-tcad/
```

Restart or reload Claude Code afterwards.

### Option C: Other agent environments

Place `SKILL.md` and `references/` in the skill, instruction, or knowledge directory supported by your agent platform. The important part is that the agent reads `SKILL.md` first and then loads reference files on demand.

If the platform does not have a formal "skill" concept, you can still use these files as project-level system instructions or knowledge documents. The minimum useful set is:

```text
SKILL.md
references/new-device-preflight.md
references/swbpy2-gsub.md
references/results-reporting.md
```

Add the SDE, SDevice, or GaN/SEB references when your task needs them.

## Prerequisites

You need your own licensed Sentaurus TCAD installation. This repository does not include Sentaurus software, licenses, official manuals, official examples, or commercial files.

Recommended setup:

- Working `swb`, `gsub`, `sdevice`, and `svisual` commands.
- A Sentaurus Workbench `STDB` directory.
- A working Sentaurus Python / `swbpy2` environment.
- Access to official Applications Library and documentation.
- Literature search tools such as Zotero, institutional access, or public databases.

On a new machine, ask the agent to run a full preflight before writing decks or submitting jobs. At minimum, it should check:

```bash
which swb gsub sdevice svisual
printf '%s\n' "$STROOT" "$STRELEASE" "$STDB"
test -n "$STDB" && test -d "$STDB" && test -w "$STDB"
```

It should also confirm that the Sentaurus license is usable, TCAD Python can import `swbpy2`, the `gsub` queue exists, SVisual/display is available, and Applications Library / manuals paths are accessible. See `references/new-device-preflight.md` for the full checklist.

If preflight fails, the agent should stop the simulation plan and report the blocker. It should not treat license, PATH, STDB, queue, or GUI failures as SDE/SDevice model problems.

## Example prompts

### First run on a new machine

```text
Use the sentaurus-tcad skill to check whether this new server can run Sentaurus simulations. Do not write decks or submit gsub yet; run the new-device preflight first and verify PATH, STROOT/STRELEASE/STDB, license, swbpy2, gsub queue, SVisual/display, manuals/examples, and STDB write permission.
```

### Create a p-GaN HEMT project from scratch

```text
Use the sentaurus-tcad skill to create a p-GaN HEMT SWB project from scratch. I want to run IdVg first to verify Vth > 1.2 V, then run BV up to 900 V. Please search official examples and references before designing the SWB tree, SDE/SDevice files, gsub submission, and result logging flow.
```

### Diagnose BV convergence failure

```text
My GaN HEMT BV node hits Step-size is too small around 720 V. The Newton maximum error coordinate is near the gate edge. Please do not blindly tune parameters; diagnose it through log → plt → tdr → reference search → fix.
```

### HeavyIon / SEB threshold scan

```text
I need a HeavyIon SEB threshold scan with LET = 0.8 pC/um and several LoadVoltage nodes. Please add experiments through SWB/swbpy2, submit them with gsub, and output curves, tables, and criteria.
```

## Quick start for new users

1. Make sure you already have a licensed, working Sentaurus TCAD environment.
2. Import `dist/sentaurus-tcad.skill` into Claude Code, or manually copy `SKILL.md` and `references/`.
3. On the first run on a new machine, ask the agent to run preflight:
   ```text
   Use the sentaurus-tcad skill to check whether this machine can run Sentaurus simulations. Do not write decks or submit gsub yet.
   ```
4. After preflight passes, describe your device, simulation targets, and required outputs.
5. After a run finishes, ask for the SWB node number, log status, `.plt` curves, `.tdr` spatial diagnosis, and a persistent report.

## What problems does it prevent?

| Problem | Skill behavior |
|---|---|
| Agent directly runs `sdevice pp*_des.cmd` | Prohibits it and requires `gsub` so SWB can track results |
| Agent only reads logs | Requires log → plt → tdr diagnosis |
| Models and parameters are guessed | Requires official examples, documentation, and literature evidence |
| Results exist only in terminal text | Requires persistent plots, tables, or reports |
| No trace of simulation iterations | Requires progress and findings notes |
| Huge monolithic skill instructions | Keeps the main skill short and moves details into references |

## Security and compliance

- This repository does not contain Synopsys Sentaurus software, licenses, official PDFs, or official example files.
- Users must ensure they have a valid Sentaurus license.
- Commands in this repository are workflow templates. Do not execute them blindly without checking paths and node numbers.
- Any destructive or overwriting operation should require backup and explicit confirmation.
- See [SECURITY.md](SECURITY.md).

## License

MIT License. See [LICENSE](LICENSE).

## Contributing

Contributions are welcome, especially:

- New references for SiC, CMOS, photonic devices, or other TCAD domains.
- More robust SVisual export templates.
- More realistic evaluation prompts.
- Installation notes for Codex, OpenCode, OpenClaw, and other agent environments.

Maintenance principle: **keep `SKILL.md` concise; put detailed domain knowledge into `references/`.**
