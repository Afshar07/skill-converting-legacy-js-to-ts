---
name: converting-legacy-js-to-ts
description: JS→TS migration in Vue/Nuxt — .vue components, composables, stores, helpers. Triggers: "convert to TS", "add types", "migrate to TypeScript", any JS→TS task.
model: haiku
---

# Converting Legacy JS to TypeScript (Vue/Nuxt)

## Overview

A conversion is a type-annotation pass, not a refactor. The converted file must be a drop-in replacement. **Backward compatibility is absolute.**

## Iron Rules

1. **Map the public surface first.** Props, emit names, slot names + slot props, `defineExpose`, module exports (default and named), keys of returned objects. In Nuxt, anything in `composables/` or `utils/` is auto-imported — absence of import statements proves nothing; grep the symbol name itself.
2. **Grep every usage before touching anything.** Types come from real call sites, not guesses. If callers pass arrays to a `rules` prop you assumed was a string, the type is `string | string[]`.
3. **Internal symbols → camelCase.** Locals, computeds, and functions used only inside this file. The file's own template counts as inside — update it in the same edit. Rename only with grep proof of zero external usage.
4. **Public surface — two regimes.**
   - **Components (`.vue`)**: props, emits, slots, and `defineExpose` keep existing names including snake_case — Vue does NOT map snake_case→camelCase. The only free equivalence is camelCase ↔ kebab-case in templates. List snake_case names in the report; rename only when the user explicitly asks.
   - **JS/TS modules**: exported functions may be renamed to camelCase via the dual-export alias bridge — `export { newCamelName as old_snake_name, newCamelName }`. Never drop the old name. Signature changes always require asking.
5. **No new `any`.** Derive from usage; use `unknown` + narrowing when genuinely unknowable; ask when stuck. Never paper over an untyped local JS dependency with a type assertion — that is fake safety. If blocked by an untyped dependency: **STOP — report it and ask the user whether to expand scope.** Do not convert any file the user has not explicitly named. Exception: a narrow cast *inside* a converted generic helper that reads dynamic keys (where the value's type is contractually tied to a parameter, e.g. a default value) is acceptable — keep such casts confined to that helper and note them in the report.
6. **Structure stays.** A `computed` stays a `computed`. No converting reactive code to module constants, no extracting helpers, no arrow-body shorthand "cleanup", no template churn (attribute reordering, self-closing tags, removing "no-op" code or blank lines). File layout is NOT structure: when consumers import through a barrel (`index.ts`), moving a function to another file is invisible to them and is the approved way to migrate large modules (see "Migrating large JS modules").
7. **Bugs get reported, not fixed.** Latent bugs, dead props, falsy-default traps: note them in your final report. Fixing changes behavior — out of scope.

## Procedure

1. **House style**: read `.ai/conventions.md` if present; otherwise match 2 already-converted files in the repo.
2. **Map public surface** (Rule 1) + **grep all usages** (Rule 2). Record findings — you'll cite them as proof.
3. **Scope is exactly one file.** Only touch the file the user explicitly named. If it depends on an untyped local `.js` helper: **STOP — report the blocker and ask the user whether to expand scope.** Never assert over untyped dependencies. Never convert a file the user did not name.
4. **Convert mechanically**:
   - `interface IProps` + `defineProps<IProps>()` + `withDefaults()` — preserve every default exactly.
   - `interface IEmits` + `defineEmits<IEmits>()`.
   - String-literal unions only where usage/lookup tables prove the full value set.
   - Naming: `I` prefix for interfaces, `T` for aliases, `E` for enums. API shapes: `I<Verb><Thing>Payload` / `I<Verb><Thing>Response`. Types go in a sibling `.types.ts`.
5. **Rename internals to camelCase** (Rule 3), including this file's template.
6. **STOP and ask**, listing each item with usage evidence, when:
   - a component's public API would change for any reason
   - a module export's *signature* would change
   - anything in a published package would change its API
7. **Verify**:
   - run `npx nuxi typecheck` or `vue-tsc --noEmit` — you are accountable for zero new errors in files you touched.
   - re-grep all usages and confirm each still matches the API.
   - for `.js` → `.ts` renames: grep import specifiers with explicit `.js` extension; removing `.js` from the specifier is the ONLY allowed edit in other files. List every such edit in the report.
8. **Report**: what was converted, internal renames (with grep proof), open questions, bugs noticed but not fixed.

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

## Migrating large JS modules (gradual extraction)

Drain function-by-function — never convert in one pass:

1. **Barrel first.** All consumers must import through a folder `index.ts`. If they import the `.js` directly, create the barrel + fix specifiers first.
2. **Staging sibling.** Create `<module>.new.ts` next to the `.js` with a brief migration header comment.
3. **Per function:** delete from `.js`, reimplement typed in `.new.ts`, dual-export:
   ```ts
   function addMemberToAccelerator(
     payload: IAddMemberToAcceleratorPayload,
   ): Promise<AxiosResponse<IApiResponse<ICreatedAcceleratorMember>>> {
     return axios_post(ROUTE_ACCELERATOR_ADD, payload);
   }
   export { addMemberToAccelerator as store_accelerator_by_root_accelerator, addMemberToAccelerator };
   ```
4. **End state.** When `.js` is empty: domain folders each contain `x.api.ts` + `x.types.ts` + `index.ts`. Top-level `index.ts` re-exports all.
5. **Verify each move.** Typecheck + grep proving both old and new name resolve from the barrel.

## Canonical pattern: dynamic-key prop readers

```ts
import { computed } from "vue";
import type { ComputedRef } from "vue";

// props: `object` not `Record<string, unknown>` — typed defineProps<IProps>() has no index signature.
// The two casts are the Rule 5 exception: confined here, type tied to defaultValue.
const computedProp = <T>(props: object, key: string, defaultValue: T): ComputedRef<T> => {
  return computed(() => {
    const source = props as Record<string, unknown> & { input?: Record<string, unknown> };
    return (source[key] || source.input?.[key] || defaultValue) as T;
  });
};
```

## Quick Reference

| Symbol | Action |
|---|---|
| local var / computed / file-private function | rename to camelCase (grep proof required) |
| camelCase prop used as kebab-case in templates | keep — Vue's free equivalence |
| snake_case prop / emit / slot | keep as-is (components); dual-export bridge (modules) |
| `defineExpose` / returned object keys | keep as-is, ASK before any change |
| module function signature | ASK — bridge covers renames, never signatures |
| default values | preserve exactly in `withDefaults` |
| unused prop/param | keep as-is, note in report |
| latent bug | keep as-is, note in report |
| untyped local `.js` dependency | STOP — report it, ask user before expanding scope |

## Rationalizations — all of these mean STOP

| Excuse | Reality |
|---|---|
| "Module constants are cleaner here" | That's a refactor. Types only. |
| "It's dead code, removing it is safe" | Churn buries the diff. Leave it, note it. |
| "The param is unused, I'll make it optional" | Signatures are public API. Ask. |
| "Worth normalizing snake_case while we're here" | Vue does NOT map snake_case→camelCase. Ask. |
| "This restructuring fixes a latent TS error" | Type what exists. If it can't be typed as-is, ask. |
| "The helper is untyped, I'll assert its return type" | Fake safety. Report it, ask the user. |
| "I need to convert the dependency first" | Scope is one file. Report the blocker, ask the user. |
| "Zero usages, renaming the prop is harmless" | Props keep their names unless the user says so. |
| "I'll rename the export, callers can update" | Never. Use the dual-export bridge. |

## Red Flags — stop and re-check the rules

- The diff touches a template line that contains no renamed internal symbol
- A `computed` disappeared from the script
- A new `?` appeared on an existing parameter, or a prop default was dropped
- `any` anywhere in the diff
- You renamed a symbol you never grepped
- Your report contains the word "also" followed by an improvement nobody asked for
- You opened or edited any file other than the one the user named
