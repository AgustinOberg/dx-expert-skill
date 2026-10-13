# Lists and performance

Read when: rendering a list of data, paginating, showing remote images, a compiler warning/bail-out,
or something feels slow.

## Lists: LegendList
* Any list of data uses `LegendList` (`@legendapp/list`) unless the project chose otherwise. Not
  `FlatList` (older code), not FlashList. A short fixed list (≤ ~20 static rows, e.g. settings)
  may be a `ScrollView` with `.map`.
* Check the LegendList docs for the installed version (`recycleItems`, `estimatedItemSize`,
  `maintainVisibleContentPosition` for chat-like lists, refresh props).
```tsx
// components/routine/routine-list.tsx
const renderItem = ({ item }: { item: Routine }) => <RoutineListItem data={item} />;

const RoutineList = () => {
  const { query, results } = useGetMyRoutines();

  const handleEndReached = () => {
    if (query.hasNextPage && !query.isFetchingNextPage) query.fetchNextPage();
  };

  return (
    <LegendList
      data={results}
      keyExtractor={(item) => item.id}
      renderItem={renderItem}
      ListHeaderComponent={RoutineListHeader}
      ListEmptyComponent={query.isPending ? RoutineListSkeleton : RoutineListEmpty}
      ListFooterComponent={query.isFetchingNextPage ? ListLoadingFooter : null}
      onEndReached={handleEndReached}
      onEndReachedThreshold={0.5}
      refreshing={query.isRefetching}
      onRefresh={query.refetch}
      contentContainerStyle={styles.content}
      testID="routine-list"
    />
  );
};
```
* `keyExtractor` always, from a stable id (never the index).
* Item, header, empty and footer are **components in their own files**, not JSX built in the parent,
  not `useMemo`/`useCallback`-wrapped JSX, not `renderHeader()` helpers.
* `renderItem` is a lowercase function that returns `<Row item={item} />`. Passing a capitalized
  component straight as `renderItem` makes the compiler treat it as a component with hooks and then the
  list calls it as a function ("Do not call Hooks inside useMemo").
* Reusable list parts take an optional `renderItem` override with a default (`references/compound-components.md`).
* Every main list has pull to refresh; paginated lists guard `fetchNextPage` as above.
* Spacing between rows comes from `contentContainerStyle` (`gap` token) or a separator component - not margins on each row.
* Rows with local state (inputs, toggles) need care with recycling: leave `recycleItems` off or key the state by item id.
* Bottom padding clears the tab bar / home indicator (`rt.insets.bottom` or the tab spacer).

## Images: expo-image through the primitive
`components/primitive/image.tsx` wraps `expo-image` (re-exporting RN `Image` loses caching - seen in production; don't).
```tsx
<Image
  source={{ uri: routine.coverUrl }}
  style={styles.cover}                 // explicit width/height or aspectRatio from tokens
  contentFit="cover"
  placeholder={{ blurhash: routine.coverBlurhash }}
  transition={200}
  recyclingKey={routine.id}            // inside recycled list rows
  accessibilityLabel={routine.name}    // or accessible={false} when decorative
/>
```

## React Compiler: keep it compiling
The compiler is on and does all memoization. A component it skips silently loses that. Rules:
* Never `// eslint-disable react-compiler`, never `'use no memo'`. A bail-out is a bug in the code: fix it.
* The compiler lint (as wired by the project's lint config) stays clean; React DevTools shows the
  "Memo ✨" badge on compiled components when you need to check one.

| Bail-out cause | Fix |
|---|---|
| Creating a hook factory inside a hook (`createFormHook()` in `useFormBuilder`) | create it once at module scope |
| Reading or writing `ref.current` during render | only in handlers and effects |
| Mutating props, state, query data or a value after it was rendered (`data.push`, `obj.x = 1`) | create new values (`[...list, item]`, `toSorted`) |
| Calling a component as a function / capitalized `renderItem` | render it as `<Row />`; lowercase render fn |
| Component defined inside another component | module scope or its own file |
| Conditional hooks, hooks in loops or after an early return | hooks at the top, unconditionally |
| Untracked observable reads (`obs.get()` in render) | `useValue(obs)`; `.peek()` in handlers |
| Dynamic hook or i18n keys built with `as any` | type the key map (`Record<Union, TranslationKey>`) |

## Performance habits
* Measure before optimizing: a React profiler (React DevTools, or Argent's `argent-react-native-profiler`
  skill when installed) for re-renders and CPU, the native profiler for hangs. No speculative optimization.
* A context whose value changes often and has many consumers re-renders all of them: split it, or move
  the hot value into a Legend observable read with `useValue` where it is drawn.
* Animations run on the UI thread with Reanimated (shared values), never by `setState` per frame.
  Motion is confined to where it adds meaning (onboarding, paywall, transitions, feedback).
* Heavy work never blocks a press: navigate first, then do slow work in the background.
* Debounce search input (`use-debounce`, ~300 ms) before it becomes a query key.
* Images sized explicitly; remote images through the primitive (disk cache by default).
* Keep JS bundles lean: no dead dependencies (one production app carried six with zero imports), no dev-only
  screens in release bundles (exclude them in Metro for production builds).
