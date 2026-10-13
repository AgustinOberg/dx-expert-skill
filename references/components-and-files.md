# Components and files

Read when: creating or splitting a component or screen, choosing where a file goes, writing props.

## Where things live
```
src/
  app/                          routes only: thin screens + _layout.tsx (no business logic)
  api/                          client.ts · dto/ · services/ · hooks/<domain>/ · mocks/ · lib/query-keys.ts
  components/
    primitive/                  thin wrappers of RN / third-party (pressable, text, image, bottom-sheet, dropdown-menu)
    ui/                         the kit: text, button, icon, icon-button, badge, skeleton, card/ …
    shared/                     app compositions: screen-header, screen-section, list-item/, floating-footer,
                                empty-state, error-state, with-layout
    <feature>/                  feature components and compound folders (kebab-case feature name)
  hooks/<domain>/useX.ts        feature, screen, state and form hooks
  state/contexts/<name>.context.tsx    feature contexts
  state/global/<name>.state.ts         Legend observables (global client state)
  config/  constants/  utils/ (pure)  types/
```
Domain folders are kebab-case (`hooks/training-plan/`); hook files are camelCase named after the hook.
Never create a parallel folder for something that already has a home (`src/contexts/` next to
`src/state/contexts/`, `hooks/trainingPlan/` next to `hooks/training-plan/` - both drifts seen in production).

## Anatomy of a component file
```tsx
// components/routine/routine-summary-card.tsx
import Card from '@/components/ui/card';
import Text from '@/components/ui/text';
import { useTranslation } from '@/hooks/i18n/useTranslation';
import type { RoutineSummary } from '@/api/dto/routine/routine.dto';
import { View } from 'react-native';
import { StyleSheet } from 'react-native-unistyles';

interface Props {
  data: RoutineSummary;
  onPress: () => void;
}

const RoutineSummaryCard = ({ data, onPress }: Props) => {
  const { t } = useTranslation();
  return (
    <Card.Root onPress={onPress} accessibilityLabel={data.name} testID={`routine-card-${data.id}`}>
      <View style={styles.meta}>
        <Card.Title>{data.name}</Card.Title>
        <Text size="sm" color="mutedForeground">
          {t('routine.exerciseCount', { count: data.exerciseCount })}
        </Text>
      </View>
    </Card.Root>
  );
};

export default RoutineSummaryCard;

const styles = StyleSheet.create((theme) => ({
  meta: { gap: theme.sizes.xs },
}));
```
* Arrow const + `export default` at the end. Primitives (`components/primitive/*`) use named exports.
* Props: `interface Props` when local; `export interface ButtonProps extends PressableProps` when the
  type is reused or the component wraps a native one. Destructure in the signature, defaults there.
* Children: `PropsWithChildren`. Slots are `React.ReactNode` props (`rightComponent`, `footer`, `prefix`, `suffix`).
* No `displayName` - names are inferred. Only an HOC or `Object.assign` needs one.
* New components take `ref` as a normal prop (React 19); no `forwardRef`.
* Types: domain types come from the DTOs (`z.infer` in `api/dto/<domain>/`); UI-only types stay
  local or go to `src/types/<domain>.ts`. Prop unions for looks come from the theme (`Color`, `Size`).
* One-line JSDoc on exported kit/shared props when the name is not enough. Comments say *why*, never *what*.

## Anatomy of a screen (the default shape)
```tsx
// app/(authenticated)/routine/[routineId].tsx
const RoutineScreen = () => {
  const { t } = useTranslation();                                     // 1. hooks
  const routineId = useRoutineId();
  const { data: routine, isPending, isError, refetch } = useGetRoutine(routineId);
  const { startSession } = useStartSession();

  const handleStart = () => {                                         // 2. handlers
    startSession(routineId);
    router.push('/session');
  };

  if (isPending) return <RoutineSkeleton />;                          // 3. early returns
  if (isError) return <ErrorState onRetry={refetch} />;

  return (                                                            // 4. sections + kit pieces
    <>
      <ScreenHeader title={routine.name} rightComponent={<RoutineOptionsButton routine={routine} />} />
      <ScrollView contentContainerStyle={styles.content}>
        <RoutineHero routine={routine} />
        <Button title={t('routine.start')} size="lg" onPress={handleStart} testID="routine-start" />
        <RoutineExercisesSection routineId={routineId} />
      </ScrollView>
    </>
  );
};

export default RoutineScreen;
```
Two shapes are current; pick by need:
* **Self-fetching sections** (default for tab roots and dashboards): the screen composes sections; each
  section calls its own data hook and returns `null` (or its empty state, per the hi-fi) when there is
  nothing. Keeps screens around 100 lines, even busy tab roots.
* **Screen hook** `useXScreen()` in `hooks/<domain>/`: when handlers coordinate several hooks (mutation +
  navigation + alert + toast). It returns data and handlers; the sections stay dumb.

Route files stay thin. A screen-scoped provider is mounted in one line:
`export default withLayout(RoutineSearchScreen, SearchRoutine.Root);`
Layouts may be `export default function XLayout()` (Expo Router convention); everything else is arrow const.

## Naming
Handlers `handleX`; navigation helpers `goToX`; module constants `UPPER_SNAKE_CASE`; label maps typed
over the union so a new member fails to compile:
```ts
const FILTERS: RoutineSearchFilter[] = ['all', 'mine', 'public'];
const FILTER_LABELS: Record<RoutineSearchFilter, TranslationKey> = {
  all: 'routineSearch.filters.all',
  mine: 'routineSearch.filters.mine',
  public: 'routineSearch.filters.public',
};
```
Domain-specific names over generic ones (`RoutineSummaryCard`, not `Card2`/`Item`).

## When to split
* A file passes ~150 lines of logic + JSX, or JSX nests more than ~4 levels → extract sections.
* The same block appears twice → one component fed by a config array. (Seen in production: a 437-line
  `personal-info.tsx` with 5 near-identical dropdown fields → one `ProfileSelectField` mapped over a `FIELDS` array.)
* A component picks between render modes from props → one component per mode:
```tsx
// BAD: one row, three behaviours picked by props
<PersonalInfoFieldRow label value editable onChangeText onPress />
// GOOD: three components that share the kit row
<InfoRow label value />
<EditableInfoRow label value onChangeText />
<PressableInfoRow label value onPress />
```
* Never define a component inside another one - it remounts on every render:
```tsx
// BAD (seen in production, card-root.tsx)
const CardRoot = ({ liquid, children }: Props) => {
  const Content = ({ children }: PropsWithChildren) => <View style={styles.card}>{children}</View>;
  return <Content>{children}</Content>;
};
// GOOD: module-scope component (or its own file), props passed explicitly
const CardSurface = ({ liquid, children }: SurfaceProps) => …;
```

## Odds and ends
* Skeletons are standalone files next to their component: `routine-skeleton.tsx` built with the kit
  `Skeleton` and the same layout tokens.
* Platform-only controls: split `x.ios.tsx` / `x.android.tsx` (or `x.tsx` fallback) instead of `Platform.OS` branches in JSX.
* Every user string through `t()`; numbers and units in the translation (`t('routine.exerciseCount', { count })`).
* No `any`, no `!`, no `as never`/`as any` casts (route strings and i18n keys are typed - fix the type).
* Remove what you don't use: commented-out JSX, `console.log`, unused imports, dead dependencies.
