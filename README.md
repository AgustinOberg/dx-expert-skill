# dx-expert

An agent skill with the front-end conventions for **Expo / React Native** apps: how components, hooks,
data, state, styling, navigation, UX and accessibility are written, plus a checklist to review the result.

## Install

```bash
npx skills add agustinoberg/dx-expert-skill
```

## What it enforces

- **Draw here, think there.** Components only draw what hooks hand them; I/O lives in plain service
  functions. Flow: screen → hook → service.
- **File layout.** One kebab-case component per file, `export default` at the end, Unistyles styles
  below it; one hook per file, named after the hook; `@/` imports.
- **Screen shape.** Hooks → handlers → early returns (loading → error → empty) → JSX of sections and kit
  pieces. No nested ternaries, no `renderX()` helpers, no components defined inside components.
- **Hook arguments.** One argument may be positional; two or more go in one named object.
- **No manual memoization.** The React Compiler is on: no `useMemo`, `useCallback` or `memo` (only a
  measured exception), no compiler opt-outs.
- **Effects are the last resort.** Derive in render, act in handlers, let queries own async work;
  `useEffect` only syncs an external system, with cleanup.
- **Compound components only when the code asks for it** (named parts, shared sibling state, create/edit
  variants, multi-step flows); a boolean that picks the rendered tree means split the component.
- **Contexts** expose `{ state, actions, meta }` with a consumer hook that throws outside its provider.
- **Data layer.** Services `Schema.parse` responses with Zod; TanStack Query with one query-key factory;
  mutations never retry; typed mocks behind the service so screens never change at the API swap.
- **State.** Server data in Query, local UI in `useState`, global/persisted client state in Legend State
  persisted with MMKV, forms in TanStack Form.
- **Styling.** Unistyles 3 theme tokens and variants only; a layered kit (primitive → ui → shared → feature).
- **Navigation and overlays.** Expo Router with typed string hrefs, `Stack.Protected` gating, one-way
  flows with `replace`; the overlay is chosen by job (modal, formSheet, bottom sheet, menu, alert, toast).
- **UX + accessibility baseline.** Keyboard never covers the input, loading/empty/error states drawn,
  44 pt targets, safe areas, haptics; role, label and state on interactive elements; `testID` on key elements.

## How it is organized

`SKILL.md` is a short core (principles, anti-patterns, review checklist). The details live in
`references/`, and the agent reads only the ones its task needs:

| Reference | Covers |
|---|---|
| `components-and-files.md` | folders, component and screen anatomy, naming, when to split |
| `compound-components.md` | when to use them, anatomy, variants, lifting providers |
| `hooks-and-effects.md` | hook kinds, arguments, memoization, the effect decision table |
| `data-layer.md` | DTOs, services, typed mocks, query keys, queries, mutations, errors |
| `state-and-forms.md` | state placement, Legend State + MMKV, contexts, flow providers, forms |
| `styling-and-kit.md` | Unistyles tokens and variants, kit layers, Pressable, icons |
| `navigation-and-overlays.md` | routes, gating, headers, overlays, back guards, one-way flows |
| `ux-keyboard-a11y.md` | keyboard, the three data states, touch targets, a11y, testID, copy |
| `lists-and-performance.md` | LegendList, images, React Compiler bail-outs, performance habits |

The skill tells the agent to read the versioned docs of the installed Expo SDK and libraries before
using an API; nothing in it pins a version.

## Review checklist

`SKILL.md` ends with a grouped checklist (structure, hooks and effects, data and state, look and
navigation, UX and a11y). The agent runs it and reports each line as ✅, ❌ or n/a before asking for approval.

## Changelog

**v2 (2026-10)**
- Short core + 9 references read on demand (v1 was one long file).
- Hook arguments: one positional allowed; two or more → one object (v1: always an object).
- No `useMemo` under the React Compiler, except a measured case (v1 allowed it for computed constants).
- Compound components only when the code asks for it (v1: the default for every screen).
- New sections: data layer, state and forms, Unistyles styling and kit, overlays by job, a11y and testID baseline.
- Keyboard via react-native-keyboard-controller and lists via LegendList.

## Compatibility

Works with any AI coding agent that supports the [Agent Skills](https://skills.sh) specification:

- Claude Code
- Cursor
- Windsurf
- Cline
- GitHub Copilot
- And 30+ more

## License

MIT
