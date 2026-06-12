---
name: converting-legacy-js-to-ts
description: Use when converting legacy JavaScript to TypeScript in Vue/Nuxt projects — .js files, composables, stores, helpers, or .vue components using <script setup> without lang="ts". Triggers include "convert to TS", "migrate to TypeScript", "add types", "make this script setup lang ts", or any JS→TS migration task.
---

# Converting Legacy JS to TypeScript (Vue/Nuxt)

## Overview

A conversion is a type-annotation pass, not a refactor. The converted file must be a drop-in replacement: every call site, template usage, and event listener keeps working with zero changes. **Backward compatibility is absolute.**

## Iron Rules

1. **Map the public surface first.** Props, emit names, slot names + slot props, `defineExpose`, module exports (default and named), keys of returned objects. In Nuxt, anything in `composables/` or `utils/` is auto-imported — absence of import statements proves nothing; grep the symbol name itself.
2. **Grep every usage before touching anything.** Types come from real call sites, not guesses. If callers pass arrays to a `rules` prop you assumed was a string, the type is `string | string[]`.
3. **Internal symbols → camelCase.** Locals, computeds, and functions used only inside this file. The file's own template counts as inside — update it in the same edit. Rename only with grep proof of zero external usage.
4. **Public surface — two regimes.**
   - **Components (`.vue`)**: props, emits, slots, and `defineExpose` keep their existing names, period — including snake_case ones. Rename only when the user explicitly requests it (then update every call site in the same pass). The ONLY free spelling equivalence is camelCase ↔ kebab-case in templates. **Vue does NOT map snake_case to camelCase** — renaming `is_icon_reversed` to `isIconReversed` silently breaks every `:is_icon_reversed` call site. Don't ask, don't suggest mid-conversion: keep the name and list it in the report.
   - **JS/TS modules**: exported functions MAY be renamed to camelCase, but only via the dual-export alias bridge — `export { newCamelName as old_snake_name, newCamelName }` — so every existing import keeps working. Never drop the old name. Signature changes (parameters added, removed, reordered, or made optional) are not covered by the bridge and still require asking the user.
5. **No new `any`.** Derive from usage; use `unknown` + narrowing when genuinely unknowable; ask when stuck. Never paper over an untyped local JS dependency with a type assertion — that is fake safety. Convert the leaf helper first. Exception: a narrow cast *inside* a converted generic helper that reads dynamic keys (where the value's type is contractually tied to a parameter, e.g. a default value) is acceptable — keep such casts confined to that helper and note them in the report.
6. **Structure stays.** A `computed` stays a `computed`. No converting reactive code to module constants, no extracting helpers, no arrow-body shorthand "cleanup", no template churn (attribute reordering, self-closing tags, removing "no-op" code or blank lines). File layout is NOT structure: when consumers import through a barrel (`index.ts`), moving a function to another file is invisible to them and is the approved way to migrate large modules (see "Migrating large JS modules").
7. **Bugs get reported, not fixed.** Latent bugs, dead props, falsy-default traps: note them in your final report. Fixing changes behavior — out of scope.

## Procedure

1. **House style**: read `.ai/conventions.md` if the repo has one; otherwise open 2 already-converted files in the same repo and match them (e.g. `interface IProps`, auto-imported `computed`).
2. **Map public surface** (Rule 1) and **grep all usages** (Rule 2). Record what you found — you'll cite it as proof.
3. **Dependency check**: if the file imports untyped local `.js` helpers whose return types you would have to assert, convert those leaf files first (same rules, recursively) or stop and report the dependency.
4. **Convert mechanically**:
   - `interface IProps` + `defineProps<IProps>()` + `withDefaults()` — preserve every default exactly.
   - `interface IEmits` + `defineEmits<IEmits>()`.
   - String-literal unions only where usage/lookup tables prove the full value set.
   - Type-only exports may be added (they are additive and erased at runtime).
   - Naming: `I` prefix for interfaces, `T` for type aliases, `E` for enums. API shapes follow `I<Verb><Thing>Payload` / `I<Verb><Thing>Response`. When migrating modules, types go in a sibling `.types.ts`, not mixed into the implementation file.
5. **Rename internals to camelCase** (Rule 3), including their uses in this file's template.
6. **STOP and ask the user**, listing each item with its usage evidence, when:
   - a component's public API would change for any reason (component naming changes only ever happen when the user asks for them)
   - a module export's *signature* would change (the alias bridge covers renames, never signatures)
   - anything in a published package (e.g. ui-layer) would change its API
7. **Verify** (file-level):
   - run the typecheck (`npx nuxi typecheck` or `vue-tsc --noEmit`) and read its output — inspection is never verification. In repos full of pre-existing errors you are accountable for one thing: zero errors in the files you touched.
   - re-grep all usages and confirm each still matches the API exactly
   - when renaming `.js` → `.ts`: grep for import specifiers with an explicit `.js` extension. Removing that `.js` from the specifier is the ONLY edit allowed in other files — nothing else on the line changes, and every such edit is listed in the report.
8. **Report**: what was converted, internal renames made (with grep proof), open questions, bugs noticed but not touched.

## Core Pattern

```ts
// BEFORE (legacy <script setup>)
const props = defineProps({
  size: { type: String, default: "md" },
  is_icon_reversed: { type: Boolean, default: false },
});

// AFTER (<script setup lang="ts">)
interface IProps {
  size?: "xs" | "sm" | "md" | "lg"; // union proven by the size→class lookup map
  is_icon_reversed?: boolean;       // snake_case kept — renaming requires asking the user
}
const props = withDefaults(defineProps<IProps>(), {
  size: "md",
  is_icon_reversed: false,
});
```

## Quick Reference

| Symbol | Action |
|---|---|
| local var / computed / file-private function | rename to camelCase (grep proof required) |
| camelCase prop used as kebab-case in templates | keep — Vue's free equivalence |
| snake_case prop/emit | type as-is, ASK before renaming |
| `defineExpose`, module exports, returned object keys | type as-is, ASK before any change |
| default values | preserve exactly in `withDefaults` |
| unused prop/param | keep as-is, note in report |
| latent bug | keep as-is, note in report |
| untyped local `.js` dependency | convert it first (leaf-up), never assert over it |

## Rationalizations — all of these mean STOP

| Excuse | Reality |
|---|---|
| "These maps never change, module constants are cleaner" | That's a refactor. Conversion adds types only. |
| "It's a no-op class / dead line, removing it is safe" | Churn buries the real diff. Leave it, note it. |
| "The param is unused, I'll make it optional" | Signatures are public API. Ask. |
| "Worth normalizing snake_case while we're here" | Vue does NOT map snake_case→camelCase. Ask, always. |
| "This restructuring fixes a latent TS error" | Type what exists. If it can't be typed as-is, ask. |
| "The helper is untyped, I'll assert its return type" | Unchecked assertion = fake safety. Convert the leaf first. |
| "Zero usages, so renaming the prop is harmless" | Public-surface renames are the user's call, not yours. |

## Red Flags — stop and re-check the rules

- The diff touches a template line that contains no renamed internal symbol
- A `computed` disappeared from the script
- A new `?` appeared on an existing parameter, or a prop default was dropped
- `any` anywhere in the diff
- You renamed a symbol you never grepped
- Your report contains the word "also" followed by an improvement nobody asked for
