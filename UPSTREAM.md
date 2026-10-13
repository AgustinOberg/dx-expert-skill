# dx-expert - upstream

Published at https://github.com/AgustinOberg/dx-expert-skill.

The app-factory copy (`.claude/skills/dx-expert/`) and the public repo hold the same files:
`SKILL.md`, `references/*.md`, `UPSTREAM.md` and `LICENSE`; the repo adds only its `README.md`.
Edit the factory copy first, then the maintainer pushes the same files to the repo (agents never run git
for it). Projects receive their own copy under `.claude/skills/dx-expert/`; edit the factory copy, never
a project copy.

## v2 vs v1
v1 was a single 886-line `SKILL.md`. v2 changes:
* **Layout:** a short core (18 principles, anti-patterns, review checklist) plus 9 references read on
  demand, instead of one long file.
* **Hook arguments:** v1 required one object even for a single argument; v2 allows one positional
  argument and requires one named object for two or more (services and helpers too).
* **Memoization:** v1 allowed `useMemo` for computed constants; v2 has no `useMemo`/`useCallback`/`memo`
  under the React Compiler (only a measured exception with a comment) and forbids compiler opt-outs.
* **Compound components:** v1 made them the preferred shape for every screen; v2 uses them only when the
  code asks for it (named parts, sibling state, create/edit variants, multi-step flows), with
  `{ state, actions, meta }` contexts and a throwing consumer hook.
* **Data layer (new):** services `Schema.parse` responses with Zod, TanStack Query with one query-key
  factory, `_def` invalidation, mutations never retry, device/SDK calls as mutations, typed mocks with
  scenarios, `{ error, code }` mapped to i18n.
* **State and forms (new):** placement table (Query / `useState` / context / flow layout / Legend State
  + MMKV) and TanStack Form with `createFormHook` created once at module level.
* **Styling (new):** Unistyles 3 tokens and variants only, a layered kit (primitive → ui → shared →
  feature) and a Pressable primitive with pressed state, haptics and a default role.
* **Navigation:** "Expo Router first" became concrete rules: global `router` + typed string hrefs,
  `Stack.Protected` gating, one-way flows with `replace`, an overlay-by-job table, unsaved-changes guard.
* **UX, lists, a11y:** keyboard via react-native-keyboard-controller (no `KeyboardAvoidingView`),
  LegendList instead of FlatList/FlashList, expo-image, and an accessibility + `testID` baseline owned
  by the kit.
* **Review:** anti-patterns taken from real production code and a grouped checklist reported as
  ✅ / ❌ / n/a before asking for approval.

License: MIT (`LICENSE`), unchanged.
