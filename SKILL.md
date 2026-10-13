---
name: dx-expert
description: Expo/React Native front-end conventions — components, hooks, React Compiler, compound components, TanStack Query + Zod + typed mocks, Legend State, Unistyles, Expo Router, overlays, UX, a11y. Use when writing, reviewing or planning mobile code; run its checklist before asking for approval.
---

# dx-expert — Expo / React Native front-end conventions

The guide for agents writing Expo / React Native front-end code. Its rules come from real production
apps and from the anti-patterns found in them. Read this core, then only the references your task
needs (table below).

**Authority, in order:** the project's recorded decisions (stack/architecture decision notes) and the
approved contract of the feature → this skill → library defaults. Rules marked
*(default — follow the project's decision if it chose otherwise)* are conventions, not fixed rules;
everything else is a rule of this skill.

**Versions:** your memory of Expo, React Native and library APIs is stale. Before using an API, read the
versioned Expo docs for the installed SDK (`https://docs.expo.dev/llms.txt`) and the library's docs for
the installed version. Install with `expo install` through the project's package manager
(`bunx expo install` with Bun). Nothing here pins a version.

## Principles

**1. Draw here, think there.** Components only draw what hooks hand them. State, data and side effects
live in hooks; I/O lives in plain service functions. Flow: screen → hook → service.

**2. Files.** One kebab-case `.tsx` per component, arrow const, `export default` at the end, Unistyles
`styles` *below* it in the same file. Compound = a folder with one file per part and the namespace in
`index.ts`. Imports use the `@/` alias (→ `src/`), never `../../`.
```tsx
const SuggestionSection = () => {
  const { t } = useTranslation();
  const { data: suggestions } = useGetMySuggestions();
  if (!suggestions?.length) return null;
  return <ScreenSection.Layout>{/* … */}</ScreenSection.Layout>;
};

export default SuggestionSection;

const styles = StyleSheet.create((theme) => ({ list: { gap: theme.sizes.md } }));
```

**3. Default screen shape.** Hooks on top → handlers (`handleX`, `goToX`) → early returns
(loading → error → empty) → JSX made of sections and kit pieces. No nested ternaries, no `renderX()`
helpers, no component defined inside another component.

**4. Hook arguments.** One argument may be positional; two or more → one named object.
```ts
useGetRoutine(routineId);                   // ok: one argument
useActivityGraph({ days, weekStartDay });   // ok: 2+ → object
useActivityGraph(days, weekStartDay);       // no
```

**5. One hook per file**, named after the hook, named export: `hooks/<domain>/useX.ts` for feature,
screen, state and form hooks; `api/hooks/<domain>/useGetX.ts` for data hooks.

**6. No manual memoization.** The React Compiler is on: no `useMemo`, `useCallback` or `memo`.
Only exception: a *measured* expensive computation, with a comment giving the measurement. If the
compiler bails out, fix the code - never `eslint-disable react-compiler` or `'use no memo'`.

**7. Effects are the last resort.** Derive during render, act in the event handler, let a query own
async work, react to writes in mutation callbacks. `useEffect` only syncs an external system, with cleanup.

**8. Compound components only when the code asks for it:** named parts that compose declaratively,
state shared by siblings, variants (create/edit) that reuse the parts, or a multi-step flow. Never
mandatory per screen. A boolean that picks *which tree* renders means split the component; booleans
for state or appearance (`isLoading`, `selected`, `disabled`) are fine.

**9. Contexts.** `createContext<T | undefined>(undefined)`; value shape `{ state, actions, meta }`;
a consumer hook that throws outside its provider; the state lives in a hook in its own file and the
provider only wires it.

**10. Data layer.** Service = plain async function that `Schema.parse`s the response (types from
`z.infer`). Data hook = TanStack Query with keys from the one key factory. Writes invalidate
`queryKeys.<domain>._def`. Mutations never retry. Device/SDK calls are mutations too. Before the
backend exists the service returns `mockResponse(...)`; hooks and screens never change at the swap.
Data hooks are UI-free: no toast, no `t()`, no router.

**11. State placement.** Server data → Query. Local UI → `useState`. Shared by siblings → context.
Multi-step flow → provider in the flow's `_layout.tsx`. Global or persisted client state → Legend
observable persisted with MMKV (rule: one mechanism, no direct MMKV calls in features). Form values →
TanStack Form, its hook created once at module level.

**12. Style = tokens + variants.** Unistyles 3 theme tokens only; kit props (`variant`, `size`,
`color`) typed from the theme; layered kit primitive → ui → shared → feature; no inline style objects
except dynamic values inside a style array.

**13. Navigation.** Global `router` + string hrefs (typed routes); `Stack.Protected` for gating; native
large titles only on tab roots; pushed screens draw the kit header; one-way flows end with `router.replace`.

**14. Overlays by job.** Route modal (a whole task) · route `formSheet` (a small navigable task) ·
TrueSheet via `useBottomSheet()` (an in-screen task) · zeego menu (options) · `Alert.alert` (destructive
confirm) · sonner toast (the result of an action).

**15. Libraries before code.** Platform → Expo Router → `@expo/ui` → the project's library catalog
(if it has one) → modules. Hand-building a sheet, menu, toast, picker, keyboard handling, list
virtualization or a gesture system needs the project owner's OK first (list the options you checked).
A new library is checked before it is proposed: maintained, New Architecture, current Expo SDK, size.

**16. UX is part of done.** The keyboard never covers the focused input; loading, empty and error
states are drawn; targets are ≥ 44 pt; safe areas respected; every overlay dismissible; every press
answers (pressed state + haptic).

**17. Accessibility + testID baseline, owned by the kit.** Role, label and state on every interactive
kit piece; a label is required on icon-only buttons; `testID` on key elements so E2E flows (Argent,
Maestro, Detox) can target them.

**18. Copy and types.** Every user-facing string goes through `t('key')` (typed resources, plurals with
`count`). No `any`, no `!` non-null assertions, no `as never`. The project's check script (types, lint,
format, tests; e.g. `bun run check`) is green.

## Anti-patterns seen in real production code (never reproduce)
1. **Component defined inside render** (`const Content = () => …` inside `CardRoot`) → remounts its
   subtree every render. Move it to module scope or its own file.
2. **`useEffect` as an event handler** (`if (error) { toast.error(…); router.back(); }`,
   `if (isGranted) router.push(…)`, `isVisible` → `present()`) → do it in the handler, the mutation
   callback, or `openSheet()`.
3. **JSX in `useMemo`/`useCallback`** (`const ListHeader = useMemo(() => <View>…</View>, [9 deps])`)
   → a component in its own file.
4. **Fire-and-forget mutation + success toast** (`deleteRoutine(id!); toast.success(…); router.back()`)
   → `await mutateAsync(…)` (or `mutate(vars, { onSuccess, onError })`), then toast and navigate; map errors.
5. **Mode-by-props component** (one row with 3 render modes chosen by `editable`/`onPress`) → 3 components.
6. **Prop drilling callbacks** (7 handlers passed into a card) → a feature context provider.
7. **Hardcoded copy** (`{count} exercises`, `'Version:'`, Spanish literals) → `t('key', { count })`.
8. **Untracked observable reads in render** (`notificationState.x.get()` in a hook body) →
   `useValue(state.x)`; `.peek()` only inside handlers.
Also seen: `createFormHook()` inside a hook (new form hook every render → compiler opt-outs);
`id!` behind `enabled`; `router.push(x as never)`; "selector" hooks over one context sold as
fine-grained; `console.log`, commented-out JSX and dead dependencies left behind.

## Read the reference you need
Paths are relative to this skill (in a project: `.claude/skills/dx-expert/references/…`).

| Your task touches… | Read |
|---|---|
| New component/screen file, props, folders, screen anatomy, splitting | `references/components-and-files.md` |
| Parts that compose (`X.Root`, `X.List`), create/edit variants, providers | `references/compound-components.md` |
| Writing a hook, arguments, any `useEffect`, memoization questions | `references/hooks-and-effects.md` |
| Services, DTOs, query keys, queries/mutations, mocks, errors, device calls | `references/data-layer.md` |
| Global/persisted state, contexts, flow providers, forms | `references/state-and-forms.md` |
| Theme, tokens, variants, kit pieces, Pressable, icons, skeleton look | `references/styling-and-kit.md` |
| Routes, layouts, headers, tabs, modals, sheets, menus, toasts, back guards | `references/navigation-and-overlays.md` |
| Keyboard, loading/empty/error states, touch targets, safe areas, a11y, testID | `references/ux-keyboard-a11y.md` |
| Lists, pagination, images, React Compiler bail-outs, performance | `references/lists-and-performance.md` |

Platform pitfalls learned the hard way (symptom → rule) live in the reference of their area, next to
the rule they explain.

## Review checklist (run it before asking for approval)
Report each line as ✅, ❌ (fixed / why not) or n/a.

**Structure**
- [ ] Components only draw; logic in hooks; I/O only in services (screen → hook → service).
- [ ] One kebab-case component per file, `export default` at the end, styles below; compound parts in a folder + `index.ts`.
- [ ] Screens follow hooks → handlers → early returns → JSX; no nested ternaries, `renderX()`, or components defined inside components.
- [ ] No mode-by-props components; no boolean that picks the rendered tree; no prop-drilled callbacks (context instead).

**Hooks & effects**
- [ ] Hook args: ≤ 1 positional, otherwise one object. One hook per file, named after it.
- [ ] No `useMemo`/`useCallback`/`memo` (or a measured one with its comment); no compiler opt-outs.
- [ ] Every `useEffect` syncs an external system and cleans up; none reacts to state to navigate, toast or open a sheet.

**Data & state**
- [ ] Responses `Schema.parse`d in the service; types from `z.infer`; keys only from the key factory.
- [ ] Writes awaited before success feedback; invalidation via `_def` (+ commented cross-domain keys); optimistic updates roll back.
- [ ] Mutations `retry: false`; device/SDK calls are mutations; data hooks contain no toast/`t()`/router.
- [ ] Errors mapped to i18n messages from `code`; global codes handled once (interceptor).
- [ ] Mocks: every hook works in the default/empty/error/slow scenarios; screens can't tell mock from API.
- [ ] State lives in the right place (Query / useState / context `{state, actions, meta}` / flow layout / Legend+MMKV / TanStack Form); observables read with `useValue`, `.peek()` in handlers.
- [ ] Forms: `createFormHook` at module level; submit disabled while invalid/submitting; dirty forms guarded on back.

**Look & navigation**
- [ ] Tokens and variants only: no hex, no raw font sizes/spacing, no inline style objects (dynamic values in arrays only); light and dark checked.
- [ ] Features use kit pieces (Pressable primitive, Text, Button, Icon, Image), never RN `Pressable`/`Image` directly.
- [ ] Global `router` + string hrefs; params typed; one-way flows use `replace`; back guard calls `proceed`.
- [ ] Right overlay for the job (modal / formSheet / TrueSheet / menu / Alert / toast); every overlay dismissible.

**UX & a11y**
- [ ] Keyboard never covers the focused input (keyboard-controller); CTA rides the keyboard when the design says so.
- [ ] Loading (skeleton), empty and error (with retry) states drawn; main lists have pull to refresh.
- [ ] Targets ≥ 44 pt (`hitSlop` on small icons); safe areas; pressed state + haptic on presses.
- [ ] Role, label, state on interactive elements; icon-only buttons labelled; `testID` on key elements.
- [ ] All copy via `t()`; no `any`, `!`, `as never`, `console.log`, commented-out code; the project's check script green.
- [ ] Matches the hi-fi design on iOS and Android; nothing hand-built that a known library (or the project's catalog) already solves.
