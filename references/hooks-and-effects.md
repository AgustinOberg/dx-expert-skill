# Hooks and effects

Read when: writing any hook, adding a `useEffect`, or tempted to memoize.

## Kinds of hooks and where they live
| Kind | Location | Name | Returns |
|---|---|---|---|
| Data (query / mutation) | `api/hooks/<domain>/useGetX.ts` | `useGetX`, `useCreateX`, `useUpdateX`, `useDeleteX` | the raw `useQuery`/`useMutation` result (infinite: `{ query, results }`) |
| Screen | `hooks/<domain>/useXScreen.ts` | `useSubscriptionScreen` | data + handlers for dumb sections |
| Feature | `hooks/<domain>/useX.ts` | `useStartSession`, `useShareLink` | an object of values and actions |
| Provider state | `hooks/<domain>/useXState.ts` | `useRoutineSearchState` | `{ state, actions, meta }` |
| Form | `hooks/<domain>/useXForm.ts` | `useCreateRoutineForm` | `{ form, … }` |
| Route params | `hooks/<domain>/useXId.ts` | `useRoutineId` | the typed param |

One hook per file, file named after the hook, **named** export (`export const useX = …`).
Hooks return objects (`{ data, handleSave }`), never positional tuples - except a `useState`-like pair.

## Arguments (rule)
One argument may be positional; two or more → one named object, typed with an interface.
```ts
export const useGetRoutine = (routineId: string | undefined) => …;      // one → positional

interface UseActivityGraphArgs {
  days: number;
  weekStartDay: WeekStartDay;
}
export const useActivityGraph = ({ days, weekStartDay }: UseActivityGraphArgs) => …;   // two → object
```
The same rule applies to service functions and helpers. Optional settings ride in the object
(`useSuggestRoutine({ targetUserId, preselected })`), never as a third positional boolean.

## Small and composed
A hook does one job. When it grows past ~60–80 lines or mixes concerns, compose smaller ones:
```ts
// hooks/training-plan/useOwnTrainingPlan.ts
export const useOwnTrainingPlan = () => {
  const { data: profile } = useGetOwnProfile();
  const plan = useGetTrainingPlanByUser(profile?.id);   // enabled only when the id exists
  return { profile, plan };
};
```
Data hooks stay UI-free; the feature or screen hook adds navigation, toasts and alerts:
```ts
// hooks/routine/useDeleteRoutineAction.ts
export const useDeleteRoutineAction = (routineId: string) => {
  const { t } = useTranslation();
  const { mutate: deleteRoutine, isPending } = useDeleteRoutine();

  // success feedback only after the server agrees - never fire-and-forget + toast
  const confirmDelete = () =>
    deleteRoutine(routineId, {
      onSuccess: () => {
        toast.success(t('routine.delete.success'));
        router.back();
      },
      onError: (error) => toast.error(t(getErrorMessageKey(error))),
    });

  const handleDelete = () =>
    Alert.alert(t('routine.delete.title'), t('routine.delete.message'), [
      { text: t('common.cancel'), style: 'cancel' },
      { text: t('common.delete'), style: 'destructive', onPress: confirmDelete },
    ]);

  return { handleDelete, isDeleting: isPending };
};
```

## No manual memoization (rule)
The React Compiler memoizes components, hooks, values and callbacks. Write plain code:
```ts
// BAD
const handlePress = useCallback(() => onSelect(item.id), [onSelect, item.id]);
const total = useMemo(() => sets.reduce((sum, s) => sum + s.reps, 0), [sets]);
export default memo(RoutineCard);
// GOOD
const handlePress = () => onSelect(item.id);
const total = sets.reduce((sum, set) => sum + set.reps, 0);
export default RoutineCard;
```
Exception: a computation that was *measured* as expensive (profiler numbers) may keep a `useMemo`
with a comment: `// measured: 18 ms per render on Pixel 6 with 2k sessions (React profiler, <date>)`.
Never memoize JSX (`useMemo(() => <Header …/>)`) - that is a component. Existing memo in old code:
leave it unless you are already rewriting that file.

## Effects: the decision table
| You want to… | Do this instead of an effect |
|---|---|
| show a value derived from props/state | compute it during render |
| react to a press / submit | the event handler |
| fetch or cache server data | a query hook |
| act after a write succeeds or fails | `await mutateAsync()` in the handler, or `mutate(vars, { onSuccess, onError })` |
| reset local state when an id changes | render the component with `key={id}` |
| open a sheet | `openSheet()` from `useBottomSheet()` in the handler |
| leave a screen when auth/onboarding changes | `Stack.Protected` guards in the layout |
| navigate after a permission is granted | the permission mutation's `onSuccess` |
| show a query error | render the error state with a retry; no auto-toast, no auto-back |
| refetch when a screen regains focus | `useFocusEffect` / `useInvalidateOnFocus({ queryKey })` |
| read a global observable | `useValue(obs)` in render, `.peek()` in handlers |

```tsx
// BAD (seen in production, routine/[routineId].tsx): an effect used as an event handler
useEffect(() => {
  if (isError) {
    toast.error(t('routine.loadError'));
    router.back();
  }
}, [isError]);
// GOOD: the screen draws the failure and lets the user act
if (isError) return <ErrorState message={t('routine.loadError')} onRetry={refetch} />;
```

## When an effect is right
Syncing with something outside React: AppState, a native event emitter, a subscription, a timer,
an imperative native API. Then:
1. keep the body small and focused, with a comment naming the external system;
2. declare every dependency (never silence the exhaustive-deps lint);
3. return the cleanup;
4. prefer `useSyncExternalStore` when you only read an external value;
5. to read the latest props inside an effect without re-subscribing, use `useEffectEvent`
   (React 19.2+, check the project's React version) instead of the "ref + effect that copies the callback" trick.
```ts
// external system: AppState
useEffect(() => {
  const subscription = AppState.addEventListener('change', (status) => setIsActive(status === 'active'));
  return () => subscription.remove();
}, []);
```

## Refs
Refs are for imperative handles (sheet, list, input focus) and values that never drive rendering.
Never read or write `ref.current` during render - the compiler bails out and the value is stale.
