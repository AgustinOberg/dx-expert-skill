# Compound components

Read when: a UI has named parts that compose (`X.Root`, `X.Input`, `X.List`), siblings share state,
the same concept has variants (create / edit), or a flow spans steps.
Background: https://www.reactjs.wiki/que-es-el-compound-components-pattern

## When - and when not
Use it **only when the code asks for it** (rule):
* named parts that the caller composes declaratively, in different orders or subsets;
* state shared by sibling parts (an input that filters a list);
* variants of one concept (create vs edit vs reschedule) that reuse the same parts;
* multi-step flows whose steps share one form or state.

Do **not** use it for: a screen without shared state (use the default screen shape), a kit piece with a
few props (`Badge`, `Divider`), or "just in case". A namespace of layout slots with no shared state is
fine and needs no context (`Card.Root / Card.Header / Card.Body`, `ScreenSection.Layout / .Header`).

## Anatomy
```
components/search/search-routine/
  index.ts                           the namespace object (default export)
  search-routine-root.tsx            Root = the provider
  search-routine-input.tsx           one file per part (+ -filter, -list, -item)
state/contexts/routine-search.context.tsx       context + Provider + throwing consumer hook
hooks/routine-search/useRoutineSearchState.ts   the state (plain hook, its own file)
```

```ts
// hooks/routine-search/useRoutineSearchState.ts
export const useRoutineSearchState = () => {
  const [searchValue, setSearchValue] = useState('');
  const [filter, setFilter] = useState<RoutineSearchFilter>('all');
  const [debouncedSearchValue] = useDebounce(searchValue, 300);
  return {
    state: { searchValue, filter, debouncedSearchValue },
    actions: { setSearchValue, setFilter },
    meta: {},
  };
};
```
```tsx
// state/contexts/routine-search.context.tsx
type RoutineSearchContextValue = ReturnType<typeof useRoutineSearchState>;
const RoutineSearchContext = createContext<RoutineSearchContextValue | undefined>(undefined);

export const RoutineSearchProvider = ({ children }: PropsWithChildren) => {
  const value = useRoutineSearchState();
  return <RoutineSearchContext.Provider value={value}>{children}</RoutineSearchContext.Provider>;
};

export const useRoutineSearch = () => {
  const context = useContext(RoutineSearchContext);
  if (!context) throw new Error('useRoutineSearch must be used within a RoutineSearchProvider');
  return context;
};
```
Write the context type explicitly (`interface RoutineSearchContextValue { state: …; actions: …; meta: … }`)
when the provider is implemented by more than one hook (variants, below).
```tsx
// components/search/search-routine/search-routine-input.tsx
const SearchRoutineInput = () => {
  const { t } = useTranslation();
  const { state, actions } = useRoutineSearch();
  return (
    <SearchInput
      value={state.searchValue}
      onChangeText={actions.setSearchValue}
      placeholder={t('routineSearch.placeholder')}
      testID="routine-search-input"
    />
  );
};

export default SearchRoutineInput;
```
```ts
// components/search/search-routine/index.ts (imports of the part files above it)
const SearchRoutine = { Root: SearchRoutineRoot, Input: SearchRoutineInput, Filter: SearchRoutineFilter, List: SearchRoutineList, Item: SearchRoutineItem };

export default SearchRoutine;
```
```tsx
// app/(authenticated)/routine-search/index.tsx - the screen composes only what it needs
const RoutineSearchScreen = () => {
  const { t } = useTranslation();
  return (
    <>
      <ScreenHeader title={t('routineSearch.title')} />
      <SearchRoutine.Input />
      <SearchRoutine.Filter />
      <SearchRoutine.List />
    </>
  );
};

export default withLayout(RoutineSearchScreen, SearchRoutine.Root);
```

## Variants: same parts, different providers
Create and edit share the parts; each brings a provider that implements the same
`{ state, actions, meta }` interface. Nothing is switched by a flag - a part that a variant does not
need is simply not rendered.
```tsx
const CreateRoutineScreen = () => (
  <CreateRoutineProvider>               {/* state from useState, submit = create */}
    <RoutineComposer.Name />
    <RoutineComposer.Exercises />
    <RoutineComposer.Submit />
  </CreateRoutineProvider>
);
const EditRoutineScreen = () => (
  <EditRoutineProvider routineId={routineId}>   {/* state seeded from the query, submit = update */}
    <RoutineComposer.Exercises />                {/* no Name part: just not rendered */}
    <RoutineComposer.Submit />
  </EditRoutineProvider>
);
```

## Lift the provider, never pass state up
When something outside the visual frame needs the state (a submit in a floating footer, a header
action), move the provider higher so it wraps both. Never `onStateChange={setParentState}`.
```tsx
<EditRoutineProvider routineId={routineId}>
  <ScreenHeader rightComponent={<RoutineComposer.SaveButton />} />
  <RoutineComposer.Exercises />
  <FloatingFooter><RoutineComposer.Submit /></FloatingFooter>
</EditRoutineProvider>
```

## Overridable parts
A part may accept an optional override with a sensible default, so another feature reuses it
(e.g. `suggest-routine` reuses `SearchRoutine.List` with a selectable row):
```tsx
interface Props {
  renderItem?: (info: { item: RoutineSearchItem }) => React.ReactElement;
}
const renderDefaultItem = ({ item }: { item: RoutineSearchItem }) => <SearchRoutineItem data={item} />;

const SearchRoutineList = ({ renderItem = renderDefaultItem }: Props) => { … };
```

## Re-renders
One context = every consumer re-renders on any change. That is fine for a screen-sized tree. Splitting
the value into `useXState / useXActions / useXMeta` hooks that read the *same* context changes nothing
(seen in production - do not). If a value changes very often and many parts read it, split it into a
second context or move it into a Legend observable read with `useValue`.

## Anti-patterns
* Booleans choosing trees: `<Composer isEditing hideClientPicker showCancel />` → variants.
* Arrays of actions with `show` conditions (`{ id: 'attach', show: !isEditing && !isForwarding }`) → render the parts.
* Seven callbacks drilled into a card (seen in production, `session-exercise-card.tsx`) → the feature provider exposes them as `actions`.
* Components defined inside `Root` → module scope.
* A compound wrapper for a screen that has no shared state → default screen shape.
