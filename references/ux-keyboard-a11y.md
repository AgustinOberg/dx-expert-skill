# UX, keyboard and accessibility

Read when: any screen with inputs, any data-driven screen, any interactive element. These are
requirements, not polish: a feature that works but breaks one of them is not done.

## Keyboard (react-native-keyboard-controller)
`KeyboardProvider` sits once in the root layout. Never RN `KeyboardAvoidingView`.
* Scrollable form → `KeyboardAwareScrollView` (scrolls the focused input into view):
```tsx
<KeyboardAwareScrollView
  bottomOffset={insets.bottom + theme.sizes.lg}
  keyboardShouldPersistTaps="handled"
  contentContainerStyle={styles.content}
>
  {/* fields */}
</KeyboardAwareScrollView>
```
* A CTA that rides on top of the keyboard → `KeyboardStickyView`, wrapped once as the shared
  `FloatingFooter` (shown only when the form has changes, if the design says so):
```tsx
<FloatingFooter>
  <form.AppForm><form.SubmitButton title={t('common.save')} /></form.AppForm>
</FloatingFooter>
```
* Multi-field forms: `returnKeyType="next"` + `onSubmitEditing` focusing the next input; the last one
  submits. `KeyboardToolbar` (prev/next/done) when the design has it.
* Lists with a search input: `keyboardShouldPersistTaps="handled"` and `keyboardDismissMode="on-drag"`.
* Closing a sheet or leaving a screen dismisses the keyboard.
Check the keyboard-controller docs for the installed version (prop names such as `bottomOffset`, `offset`).

## The three states of every data-driven screen
| State | Draw | Notes |
|---|---|---|
| Loading | the component's skeleton (`<name>-skeleton.tsx`, same layout) | never a blank screen; a spinner only inside small controls |
| Error | the shared `ErrorState` with the mapped message and a retry (`refetch`) | never `null`, never an auto-toast + auto-back |
| Empty | the shared `EmptyState`: copy that says why it's empty + the next action | a self-fetching section may return `null` only if the hi-fi says so |
```tsx
if (isPending) return <RoutineListSkeleton />;
if (isError) return <ErrorState message={t(getErrorMessageKey(error))} onRetry={refetch} />;
if (!routines.length) return <EmptyState title={t('routines.empty.title')} action={<CreateRoutineButton />} />;
```
Every state must be reachable on the simulator through the mock scenarios (default/empty/error/slow).
Main lists get pull to refresh. Writes show progress on the button (`isLoading`, disabled) and their
result as a toast only after they settle.

## Feedback
* Every press answers: pressed state + light haptic, both from the Pressable primitive.
* Key outcomes (purchase, finished session, error on submit) add a notification haptic.
* Motion is purposeful and short; respect reduced motion (`useReducedMotion()` from Reanimated → skip or
  shorten the animation).

## Touch targets and safe areas
* Every interactive element ≥ 44 × 44 pt; small icons get `hitSlop` to reach it. Keep 8 pt between targets.
* Safe areas: `rt.insets` in the stylesheet or `useSafeAreaInsets()`; nothing under the notch, status bar,
  home indicator or tab bar. Tab-root scroll views use `contentInsetAdjustmentBehavior="automatic"`;
  don't pin content under native tabs with `flexGrow` + `marginTop: 'auto'`.
* Content that can grow scrolls; nothing is clipped without a way to reach it. Text truncates with
  `numberOfLines` only where the design shows it.

## Dismissibility and doors
* Every modal, sheet and menu closes by gesture and/or a visible close button; destructive actions confirm first.
* Back behaves as the flow says: one-way doors use `router.replace` and show no back affordance.
* A dirty form is guarded on back (`references/navigation-and-overlays.md`).

## Accessibility baseline (rule - the kit owns it)
| Element | Required |
|---|---|
| Any pressable | `accessibilityRole` (default `button` in the primitive; `link`, `tab`, `checkbox`, `switch` when true) |
| Icon-only button | `accessibilityLabel` - required by the `IconButton` type, not optional |
| Stateful control | `accessibilityState`: `disabled`, `selected`, `checked`, `busy`, `expanded` as they apply |
| Input | a visible label wired as `accessibilityLabel`; error text announced with the field |
| List row / card | one focus stop: `accessible` on the container with a combined label (title + key meta) |
| Decorative image/icon | `accessible={false}`; meaningful images get a label |
| Non-obvious outcome | `accessibilityHint` ("Deletes the routine") |
```tsx
// BAD (seen in production, secondary-stack.tsx): the back chevron has no role and no label
<Pressable onPress={goBack}><ChevronLeft … /></Pressable>
// GOOD
<IconButton icon="ChevronLeft" accessibilityLabel={t('common.back')} onPress={goBack} testID="header-back" />
```
* Text scales with the system setting: never `allowFontScaling={false}`; cap with
  `maxFontSizeMultiplier` only on dense UI (tab labels, badges) and check the screen at the largest size.
* Contrast comes from the tokens; never signal state by colour alone (add an icon or text).
* Labels and hints are translated copy (`t()`), like everything else.

## testID baseline (for E2E flows: Argent, Maestro, Detox)
Put a stable `testID` on: each screen's root, primary CTAs, form inputs and submit, list items
(`${prefix}-${item.id}`), tabs, header actions, sheet roots and their confirm buttons, and the
loading/empty/error states. Format: kebab-case `<feature>-<element>` (`routine-start`,
`routine-card-${id}`, `routine-name-input`) *(default — follow the project's decision if it chose otherwise)*.
Kit pieces forward `testID` to the native element; never derive it from translated text.

## Copy
Every user-facing string, label and hint goes through `t('key')`, in the product's languages and
pronoun convention (check the project's i18n decision before writing a string; the language you talk
to the user in is not the product's). Plurals and numbers via `t('key', { count })`; dates via
`Intl`/`dayjs` with the user's locale. Translations live in one file per feature and language
(`languages/<lang>/<feature>.json`) merged by an index, so parallel work never edits the same file.
