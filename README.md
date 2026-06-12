# converting-legacy-js-to-ts

A [Claude Code agent skill](https://agentskills.io/specification) that enforces safe, backward-compatible conversion of legacy JavaScript to TypeScript in Vue/Nuxt projects:

- `.vue` components: `<script setup>` → `<script setup lang="ts">` with `interface IProps` / `IEmits`
- plain `.js` files: composables, stores, helpers → `.ts`

The skill's core guarantee: **a conversion is a type-annotation pass, not a refactor.** Converted files are drop-in replacements; the public surface (props, emits, slots, exports, returned keys) never changes without explicit user approval.

Built test-first: baseline agents were run *without* the skill, their violations documented verbatim, and every rule in `SKILL.md` counters an observed failure.

## Install

Clone into your personal skills directory:

```bash
git clone <this-repo-url> ~/.claude/skills/converting-legacy-js-to-ts
```

On Windows:

```powershell
git clone <this-repo-url> "$env:USERPROFILE\.claude\skills\converting-legacy-js-to-ts"
```

Claude Code picks it up automatically; it triggers on JS→TS conversion/migration requests.

## Contents

- `SKILL.md` — the skill: iron rules, procedure, quick reference, rationalization counters, red flags.
