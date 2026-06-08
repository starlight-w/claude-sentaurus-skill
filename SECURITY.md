# Security Policy

## Scope

This repository contains an Agent Skill for Sentaurus TCAD workflows. It contains Markdown instructions, references, evaluation prompts, and a packaged `.skill` artifact.

It does **not** contain:

- Synopsys Sentaurus software binaries.
- Sentaurus license files or license server configuration.
- Official Synopsys manuals or Applications Library projects.
- API keys, tokens, passwords, or private project data.

## Responsible use

The skill is designed to make TCAD workflows more reproducible and safer, but it can still cause problems if an agent blindly executes commands in the wrong directory or node.

Before running commands, users and agents should verify:

- The active Sentaurus environment: `STROOT`, `STRELEASE`, `STDB`.
- The project path and SWB node numbers.
- The number of currently running `sdevice` jobs.
- Whether any command may overwrite existing project files.

## Destructive operations

This skill should not require destructive filesystem operations. If a future contribution adds scripts or instructions that delete, overwrite, or move project data, they must:

1. Explain exactly what will be changed.
2. Create a backup when practical.
3. Ask for explicit user confirmation.
4. Avoid broad delete commands such as recursive project deletion.

## Command injection

When adapting examples into automation scripts, do not concatenate untrusted user input into shell commands. Prefer structured argument lists in Python and validate node numbers and paths before invoking `gsub` or other tools.

## Proprietary content

Do not contribute proprietary Synopsys files, copied manual pages, full Applications Library projects, license server details, or private simulation decks unless you have the legal right to publish them. Short command snippets and workflow descriptions are acceptable.

## Reporting issues

If you find a security or compliance issue, please open a GitHub issue with:

- The affected file.
- The specific risky content.
- Why it is risky.
- A suggested fix if possible.
