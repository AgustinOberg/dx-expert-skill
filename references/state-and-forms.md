# State and forms

Read when: deciding where a piece of state lives, writing a context or a global store, building a form or a multi-step flow.

## Where state lives
| State | Home |
|---|---|
| Server data (anything a request returns) | TanStack Query - never copied into `useState` or a store |
| Local UI of one component (open, selected tab, input text) | `useState` in the component or its hook |
| Shared by sibling parts of one feature | context provider, value `{ state, actions, meta }` |
| Shared by the steps of a flow | provider mounted in the flow's `_layout.tsx` |
| Global or persisted client state (prefs, session flags, active session) | Legend observable in `state/global/<name>.state.ts`, persisted to MMKV (rule) |
| Form values, validation, submit | TanStack Form (`useAppForm`) |
| Identity of what is shown (ids, slugs) | route params |
| Tokens and secrets | `expo-secure-store` (refresh token); access token in memory only |

## Global client state: Legend observable + MMKV
```ts
// state/global/calendar.state.ts
interface CalendarState {
  weekStartDay: WeekStartDay;
}

export const calendarState = observable<CalendarState>({ weekStartDay: 1 });

syncObservable(calendarState, {
  persist: { name: '@@state/calendarState', plugin: ObservablePersistMMKV },
});
```
* Read in render with a tracking hook: `const weekStartDay = useValue(calendarState.weekStartDay);`
* Write with `.set()`: `calendarState.weekStartDay.set(0)`; actions may live in the observable or as functions in the same file.
* Read inside handlers with `.peek()` (no subscription): `if (sessionState.status.peek() === 'IN_PROGRESS') …`
* Never `.get()` in a render body or hook body - it does not subscribe, so the UI goes stale.
* Persisted `Date`s need a `transform.load` to rehydrate. Persist names follow `@@state/<name>`.
* MMKV is used through Legend's persist plugin, never raw in features (rule). `src/config/storage.ts`
  holds only the MMKV instance the persist plugin and non-React code (e.g. the API client) share.
* Logout resets every user-scoped observable together with `queryClient.clear()`.
Check the Legend State docs for the installed version: v3 (`useValue`) is published under the npm `beta` tag while `latest` is v2
(`useSelector(() => state.x.get())`); hook names also changed across v3 releases.

## Feature context: `{ state, actions, meta }` (rule)
```tsx
// state/contexts/routine-edition.context.tsx
export interface RoutineEditionContextValue {
  state: { exercises: TrainingExercise[]; isDirty: boolean };
  actions: { addExercise: (exercise: Exercise) => void; removeExercise: (exerciseId: string) => void; save: () => void };
  meta: { routine?: Routine; isLoading: boolean; isError: boolean; isSaving: boolean };
}

const RoutineEditionContext = createContext<RoutineEditionContextValue | undefined>(undefined);

export const RoutineEditionProvider = ({ children }: PropsWithChildren) => {
  const routineId = useRoutineId();
  const value = useRoutineEditionState(routineId);   // the logic lives in hooks/routine/useRoutineEditionState.ts
  return <RoutineEditionContext.Provider value={value}>{children}</RoutineEditionContext.Provider>;
};

export const useRoutineEdition = () => {
  const context = useContext(RoutineEditionContext);
  if (!context) throw new Error('useRoutineEdition must be used within a RoutineEditionProvider');
  return context;
};
```
* `state` = what parts draw; `actions` = what parts call; `meta` = status and read-only context
  (the provider's own queries: `isLoading`, `isError`; mode; ids).
* The provider is thin; the state hook (its own file) owns `useState`, queries and mutations.
* The consumer hook lives in the `.context.tsx` file and throws outside the provider.

## Flow providers in `_layout.tsx`
Every step screen of a multi-step flow shares one provider mounted by the flow's layout:
```tsx
// app/(authenticated)/(onboarding)/_layout.tsx
const OnboardingLayout = () => (
  <OnboardingProvider>
    <Stack screenOptions={{ headerShown: false }} />
  </OnboardingProvider>
);

export default OnboardingLayout;
```
The last step submits through the provider; leaving the flow uses `router.replace` (or flips the
`Stack.Protected` guard), so back never returns into a finished flow.

## Forms: TanStack Form, hook created ONCE at module level
```ts
// components/form/form-context.ts
export const { fieldContext, formContext, useFieldContext, useFormContext } = createFormHookContexts();

// components/form/form.ts - module scope, imported everywhere
export const { useAppForm, withForm } = createFormHook({
  fieldComponents: { TextField, TextArea, Switch },
  formComponents: { SubmitButton },
  fieldContext,
  formContext,
});
```
Two files avoid an import cycle (field components import `useFieldContext`). Location
`components/form/` *(default — follow the project's decision if it chose otherwise)*; the rule "created once, at module level" is fixed.
```ts
// BAD (seen in production, hooks/useFormBuilder.ts): a new form hook on every render → 18 compiler opt-outs
export const useFormBuilder = () => createFormHook({ fieldComponents: { … }, fieldContext, formContext });
```
Field components read the field from context and own label, error and a11y:
```tsx
// components/form/text-field.tsx
const TextField = ({ label, ...props }: Props) => {
  const field = useFieldContext<string>();
  const error = field.state.meta.isTouched ? field.state.meta.errors[0]?.message : undefined;
  return (
    <TextInput
      label={label}
      value={field.state.value}
      onChangeText={field.handleChange}
      onBlur={field.handleBlur}
      error={error && translateValidation(error)}   // schema messages are i18n keys
      {...props}
    />
  );
};
```
The form hook owns `useAppForm` + the mutation + navigation + feedback:
```ts
// hooks/routine/useCreateRoutineForm.ts
export const useCreateRoutineForm = () => {
  const { t } = useTranslation();
  const { mutateAsync: createRoutine } = useCreateRoutine();
  const form = useAppForm({
    defaultValues: { name: '' },
    validators: { onChange: createRoutineFormSchema },   // Zod schema at module scope, messages = i18n keys
    onSubmit: async ({ value }) => {
      try {
        const { id } = await createRoutine(value);       // awaited: isSubmitting covers the request
        router.replace(`/routine/${id}`);                // one-way: back never returns to the form
      } catch (error) {
        toast.error(t(getErrorMessageKey(error)));
      }
    },
  });
  return { form };
};
```
```tsx
// in the screen
<form.AppField name="name">
  {(field) => <field.TextField label={t('routine.create.name')} autoFocus testID="routine-name-input" />}
</form.AppField>
<form.AppForm>
  <form.SubmitButton title={t('common.create')} testID="routine-create-submit" />
</form.AppForm>
```
* `SubmitButton` subscribes to `canSubmit` / `isSubmitting` itself (disabled + loading); screens never track that by hand.
* Show a floating save only when the values differ from the defaults:
  `<form.Subscribe selector={(s) => !s.isDefaultValue}>{(hasChanges) => …}</form.Subscribe>`.
* Validation runs through the form `validators` (`onChange`, plus `onBlur`/`onMount` when the design needs it), not `safeParse` inside `onSubmit`.
* When fields and submit live in different components (submit in a footer outside the frame), a provider
  holds the form: `{ state: { form }, actions: { submit }, meta: { isSubmitting } }`.
* A dirty form is guarded on back (`references/navigation-and-overlays.md`) and sits in a keyboard-aware
  container (`references/ux-keyboard-a11y.md`).
* Check the TanStack Form docs for the installed version (composition API, error shape).
