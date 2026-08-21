# elux-automation-skills

Reusable **AI agent skills** for the eLux UI automation project -- reusable
step-by-step playbooks that can be handed to an AI coding agent (GitHub
Copilot CLI, Claude Code, or any other agent that supports the same
`SKILL.md` convention) to make it more effective at working with:

- [elux-automation-server](https://github.com/blehalogika/elux-automation-server) --
  the Flask HTTP API that runs on the eLux device itself.
- [elux-automation-client](https://github.com/blehalogika/elux-automation-client) --
  the Page Object Model / `e2e_tests/` suite that drives it.

This repo is deliberately kept **separate** from both of those: a skill is
agent-facing documentation/process, not application code, so it has its own
lifecycle (versioned, reused, and updated independently of either repo's
release cadence) and can be handed out (copied, cloned, submoduled) without
pulling in either codebase.

## Layout

Each top-level folder is one self-contained skill, matching the layout a
`~/.copilot/skills/<name>/SKILL.md` (or Claude Code's `.claude/skills/<name>/SKILL.md`)
directory expects -- so a whole skill folder can be copied in as-is:

```
elux-ui-explorer/
    SKILL.md    Screenshot + AT-SPI tree exploration workflow for writing a
               NEW Locator/Page Object/e2e_tests test against elux-automation-client,
               without SSH/shell access to the eLux device -- see that file for
               the full step-by-step workflow.
```

## Using a skill

**GitHub Copilot CLI:** copy (or symlink) the skill folder into your user
skills directory, then it's picked up automatically:
```powershell
# PowerShell
Copy-Item -Recurse .\elux-ui-explorer "$env:USERPROFILE\.copilot\skills\elux-ui-explorer"
```
```bash
# Linux/macOS
cp -r elux-ui-explorer ~/.copilot/skills/elux-ui-explorer
```

**Claude Code:** copy the same folder into `.claude/skills/` (either a
project-local `.claude/skills/` inside `elux-automation-client`/
`elux-automation-server`, or your user-level Claude Code skills directory).

**Any other agent:** `SKILL.md` is plain Markdown with a small YAML
frontmatter header (`name`, `description`, `user-invocable`) -- point any
agent that reads project instructions at the file directly if it doesn't
have a dedicated skills mechanism.

## Adding a new skill

Create a new top-level folder named after the skill (kebab-case), with a
`SKILL.md` inside using the same frontmatter shape as
`elux-ui-explorer/SKILL.md`:
```yaml
---
name: my-new-skill
description: >-
    One or two sentences describing what this does and, critically, WHEN an
    agent should use it ("Use when the user asks to ...").
user-invocable: true
---
```
Keep each skill scoped to one coherent workflow (not a grab-bag of
unrelated tips) -- `elux-ui-explorer` is a good template: a numbered,
step-by-step loop with concrete commands/examples at each step.
