# Navigation and overlays

Read when: adding routes or layouts, headers, tabs, modals, sheets, menus, toasts, confirmations, back
guards or one-way flows. Expo Router changes every SDK: read the versioned docs for the installed SDK
(`https://docs.expo.dev/llms.txt`) before using an API (e.g. NativeTabs moved import paths across SDKs).

## Navigating
* Import the global `router` from `expo-router`; `useRouter()` is not needed.
* String hrefs, checked by typed routes: `` router.push(`/routine/${routine.id}`) ``. No `{ pathname, params }`
  objects, no `as never` casts - a cast hides a broken route.
* `push` to go deeper, `back` to return, `replace` to leave something that must not come back (below).
* `useNavigation()` only for React Navigation features Expo Router doesn't wrap (e.g. `usePreventRemove`).
* Params: `useLocalSearchParams<{ routineId: string }>()`, wrapped in a hook when several files need it:
```ts
// hooks/routine/useRoutineId.ts
export const useRoutineId = () => useLocalSearchParams<{ routineId: string }>().routineId;
```
* Data that identifies a screen travels in the URL (ids, slugs), never through context or a store.

## Layouts and gating
```tsx
// app/_layout.tsx (sketch) - providers once, then guarded groups
<QueryClientProvider client={queryClient}>
  <KeyboardProvider>
    <Stack screenOptions={{ headerShown: false }}>
      <Stack.Protected guard={isAuthenticated}>
        <Stack.Screen name="(authenticated)" />
      </Stack.Protected>
      <Stack.Protected guard={!isAuthenticated}>
        <Stack.Screen name="auth" />
      </Stack.Protected>
    </Stack>
    <Toaster />
  </KeyboardProvider>
</QueryClientProvider>
```
* Auth and onboarding are `Stack.Protected` guards. Logging in or finishing onboarding flips the state;
  the router moves by itself - never `router.push` after login, never a redirect effect.
* Groups reachable by deep link pin their base screen so back works:
  `export const unstable_settings = { initialRouteName: '(tabs)' };`
* Incoming links that need rewriting go through `app/+native-intent.tsx`.
* Tabs: `NativeTabs` with SF Symbols and an Android fallback icon (import path per SDK).
* Flow providers wrap the flow's `<Stack />` in its `_layout.tsx` (`references/state-and-forms.md`).

## Headers
* `headerShown: false` app-wide. Pushed screens draw the kit header
  (`<ScreenHeader title rightComponent />`).
* Native large titles **only on tab roots**: `headerLargeTitle` in the tab's stack layout, the scroll view
  with `contentInsetAdjustmentBehavior="automatic"`, and `useScrollToTop(ref)` so tapping the tab scrolls up.
* Native chrome keeps the system font and system colours (custom `fontFamily` blanked the large title on
  iOS 26; colours read from the theme went stale after an appearance switch); drive the native scheme
  with `Appearance.setColorScheme`.
* Changing app-wide chrome: build ONE screen, show it on iOS and Android, get the project owner's OK,
  then roll out.

## Choosing the overlay
| The user needs to… | Use | Notes |
|---|---|---|
| do a whole task with its own steps (create routine, training session) | route with `presentation: 'modal'` / `'fullScreenModal'` | own stack inside; params in the URL |
| make a small navigable choice (picker, options, sign-in) | route with `presentation: 'formSheet'` + `sheetAllowedDetents` | the default first choice for sheets; deep-linkable |
| do an in-screen task tied to this screen's state (edit a note, multi-select, report) | TrueSheet primitive via `useBottomSheet()` | opened imperatively, no URL |
| pick from a list of actions on an item | zeego menu (`primitive/dropdown-menu`) | destructive items flagged `destructive` |
| confirm something destructive or irreversible | `Alert.alert` with `style: 'destructive'` + a cancel button | |
| learn the result of an action | `toast.success` / `toast.error` (sonner-native, `<Toaster />` in the root layout) | only after the write settled |

Every overlay is dismissible (swipe and/or a visible close); a sheet's dismiss also dismisses the keyboard.

## Component sheets: `useBottomSheet()` + `sheetRef`
```ts
// hooks/useBottomSheet.ts
export const useBottomSheet = () => {
  const sheetRef = useRef<BottomSheetRef>(null);
  const openSheet = () => sheetRef.current?.present();
  const closeSheet = () => sheetRef.current?.dismiss();
  return { sheetRef, openSheet, closeSheet };
};
```
```tsx
// the screen owns the handle; the sheet receives it as a prop
const { sheetRef, openSheet } = useBottomSheet();
<IconButton icon="Flag" accessibilityLabel={t('routine.report')} onPress={openSheet} />
<ReportRoutineSheet sheetRef={sheetRef} routineId={routineId} />
```
The sheet component renders the `BottomSheet` primitive (TrueSheet, `detents={['auto']}` - prop names
per installed version) with `ref={sheetRef}`. Never an `isVisible` prop plus an effect that calls `present()`.

## Unsaved-changes guard
`usePreventNavigationBack({ shouldPrevent, onBackAttempt })` (in `hooks/navigation/`) wraps React
Navigation's `usePreventRemove`; `onBackAttempt` receives `proceed`, which re-dispatches the blocked
action (`navigation.dispatch(data.action)`). Older versions took two positional args - new code
uses the object (hook-args rule).
```ts
// BAD (seen in production, personal-info.tsx): router.back() is blocked again while hasChanges is true → loop
usePreventNavigationBack(hasChanges, () => showModal(() => router.back()));

// GOOD: confirm with proceed; read live state so save-then-leave never prompts
usePreventNavigationBack({
  shouldPrevent: hasChanges,
  onBackAttempt: (proceed) => {
    if (form.state.isDefaultValue) return proceed();   // just saved and reset → leave silently
    confirmDiscard(proceed);                            // Alert: cancel | destructive "Discard" → proceed
  },
});
```
After a successful save, `form.reset(savedValues)` before `router.back()`.

## One-way flows
Places the user must not return to with back: a finished create form, a completed onboarding step,
a submitted purchase, a closed session.
* Leave with `router.replace(...)` (create → `` router.replace(`/routine/${id}`) ``), or flip a
  `Stack.Protected` guard; `router.dismissTo(...)` / `dismissAll()` to unwind a modal stack (check the SDK docs).
* No back affordance on a one-way screen; gestures disabled when back would break state (`gestureEnabled: false`).
* Navigate first, then do slow background work; don't make the user wait on the network to leave.
