# Sentaurus TCAD End-to-End Agent Skill

[中文 README](README.md) | English README

This repository provides an **end-to-end Sentaurus TCAD simulation skill** for agent environments such as Claude Code, Claude.ai, OpenAI Codex, OpenCode, and OpenClaw.

It is not a replacement for Sentaurus TCAD and does not include any Synopsys proprietary files. Instead, it provides workflow instructions and safety constraints that help an AI agent run TCAD work in a more reliable, traceable, and research-grade way.

The intended workflow is:

```text
Problem definition → literature and documentation search → official examples check → SWB project tree → SDE/SDevice/SVisual setup → gsub submission → monitoring → log/plt/tdr diagnosis → visualization and report → knowledge capture → next iteration
```

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

## Installation

### Option A: Import the `.skill` package

If your agent environment supports `.skill` packages, import:

```text
dist/sentaurus-tcad.skill
```

### Option B: Manual installation for Claude Code

```bash
mkdir -p ~/.claude/skills/sentaurus-tcad
cp SKILL.md ~/.claude/skills/sentaurus-tcad/
cp -r references ~/.claude/skills/sentaurus-tcad/
```

Restart or reload Claude Code afterwards.

### Option C: Other agent environments

Place `SKILL.md` and `references/` in the skill, instruction, or knowledge directory supported by your agent platform. The important part is that the agent reads `SKILL.md` first and then loads reference files on demand.

## Prerequisites

You need your own licensed Sentaurus TCAD installation. This repository does not include Sentaurus software, licenses, official manuals, official examples, or commercial files.

Recommended setup:

- Working `swb`, `gsub`, `sdevice`, and `svisual` commands.
- A Sentaurus Workbench `STDB` directory.
- A working Sentaurus Python / `swbpy2` environment.
- Access to official Applications Library and documentation.
- Literature search tools such as Zotero, institutional access, or public databases.

On a new machine, ask the agent to check:

```bash
which swb gsub sdevice svisual
printf '%s\n' "$STROOT" "$STRELEASE" "$STDB"
```

## Example prompts

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
