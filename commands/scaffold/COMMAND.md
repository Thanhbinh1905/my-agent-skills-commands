---
name: scaffold
description: Scaffold a new OpenCode skill or command.
usage: /scaffold skill <name> | /scaffold command <name>
---

You are scaffolding a new OpenCode resource.

The first argument ($1) is the resource type.
The second argument ($2) is the resource name.

Supported types:
- skill
- command

Follow the instructions strictly.

---

## If type is "skill"

Create the following directory:

.opencode/skills/$2/

Create this file:

.opencode/skills/$2/SKILL.md

Use the template below:

---
name: $2
description: Ask the user for a short description, or infer from the name.
---

## What I do

- Describe what this skill does.
- Explain how it modifies agent behavior.

## When to use me

Describe trigger scenarios when this skill should activate.

## Constraints

- Do not perform unrelated actions.
- Keep scope focused.

## Examples

Example prompts that should activate this skill.

---

Rules:
- Folder name MUST match skill name.
- Use lowercase kebab-case.
- No spaces or underscores.
- No leading or trailing hyphens.


---

## If type is "command"

Create the following directory:

.opencode/commands/$2/

Create this file:

.opencode/commands/$2/COMMAND.md

Use this template:

---
name: $2
description: Ask the user what this command should do.
usage: /$2
---

You are executing the "$2" command.

## Behavior

Describe what this command does.

## Steps

1. Understand user intent.
2. Perform required workflow.
3. Produce structured output.

## Output expectations

- Be concise.
- Follow repository conventions.

---

Rules:
- Command name must be lowercase kebab-case.
- Directory name must equal command name.
- Always ask for clarification if the name is ambiguous.
