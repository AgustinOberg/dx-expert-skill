# Data layer

Read when: writing DTOs, services, mocks, query keys, query/mutation hooks, error handling, device/SDK calls.
Flow: **screen → hook → service**. Screens never import services; services never import React.

## Layout
```
src/api/
  client.ts                       axios instance `api` + interceptors, getApiErrorCode, isNetworkError
  dto/<domain>/*.dto.ts           <domain>.dto.ts (responses) · <verb>-<domain>.dto.ts (requests)
  services/<domain>Service.ts     plain async functions, one file per domain
  mocks/mock.ts · mocks/<domain>.fixtures.ts     mockResponse() + scenario · typed fixtures
  hooks/<domain>/useGetX.ts       one query/mutation hook per file
  lib/query-keys.ts · lib/errors.ts              the one key factory (owned by the contract) · code → i18n key
```
File names (rule): `services/<domain>Service.ts` with exported async functions; names in the feature's approved contract win.

## DTOs = the API contract
```ts
// api/dto/routine/routine.dto.ts
export const routineSchema = z.object({
  id: z.string(),
  name: z.string(),
  exerciseCount: z.number().int(),
  updatedAt: z.coerce.date(),
});
export type Routine = z.infer<typeof routineSchema>;
```
Schemas `camelCaseSchema`, types `PascalCase` (`Routine`, `CreateRoutineDto`); never a hand-written
interface beside a schema. Lists answer `{ data, page, limit, hasNext }` (one shared `pageSchema(item)`
helper). Validation messages are i18n keys, never literal copy.

## Services: parse at the boundary
```ts
// api/services/routineService.ts
export const getRoutine = async (routineId: string): Promise<Routine> => {
  const res = await api.get(`/routine/${routineId}`);
  return routineSchema.parse(res.data);
};

export const createRoutine = async (body: CreateRoutineDto) => {
  const res = await api.post('/routine', body);
  return idResponseSchema.parse(res.data);           // writes answer { id } or { success }
};
```
Named arrow exports; same argument rule as hooks; no React, toasts, navigation or side effects beyond
the request (seen in production: a `getProfile()` that also configured IAP - don't). Pagination in `params`, never headers.

## Typed mocks behind the service (before the backend exists)
The `screen → hook → service` chain exists from day one; before the backend exists only the service
body is fake. Same signature as the real service, so hooks and screens never change at the swap:
```ts
// api/mocks/mock.ts - sketch
export const mockScenarioSchema = z.enum(['default', 'empty', 'error', 'slow']);
/** Resolves like a request: waits, then returns the scenario's fixture or throws an API-shaped error. */
export const mockResponse = async <T>({ data, empty }: { data: T; empty: T }): Promise<T> => { … };

// api/services/routineService.ts
export const getRoutines = () => mockResponse({ data: routineStore.list(), empty: [] });            // mocks
export const getRoutines = async () => routineListSchema.parse((await api.get('/routine')).data);   // API
```
* Fixtures `satisfies Routine[]`, realistic (long names, zero/max values, missing optionals). The error
  scenario throws what the real client throws (`{ error, code }` body), so error mapping is unchanged.
* Writes mutate an in-memory store in `mocks/`; the scenario is switched from a dev-only screen
  (persisted in MMKV, excluded from release bundles). Device capabilities that *are* the experience (camera,
  picker, haptics) stay real; anything that needs a server is mocked.

## Query keys
One `createQueryKeyStore` (`@lukemorales/query-key-factory`); never a raw array key in a hook.
```ts
export const queryKeys = createQueryKeyStore({
  routine: { detail: (routineId: string) => [routineId], search: (params: RoutineSearchParams) => [params], mine: null },
  exercise: { list: (language: string) => [language] },   // localized content: language is part of the key
});
```
Parts are bare (the store already prefixes `routine`). A new key is a contract change: report it; whoever
owns the contract (not a feature task) edits `query-keys.ts`.

## Query hooks
```ts
// api/hooks/routine/useGetRoutine.ts
export const useGetRoutine = (routineId: string | undefined) =>
  useQuery({
    ...queryKeys.routine.detail(routineId ?? ''),
    queryFn: () => getRoutine(routineId ?? ''),
    enabled: !!routineId,                        // never `routineId!`
  });
```
Infinite lists (`useInfiniteQuery`, `initialPageParam: 0`, `getNextPageParam: (last) => (last.hasNext ?
last.page + 1 : null)`, `placeholderData: keepPreviousData` for search) return `{ query, results }` with
`const results = query.data?.pages.flatMap((page) => page.data) ?? [];` in the hook body (no `useMemo`).
Screens read `isPending` / `isError` / `data` / `refetch`; a query that can be disabled stays pending
forever, so its skeleton uses `isLoading`.

## Query client and freshness
`src/config/query-client.ts`: module-scope client, `staleTime: Infinity`, queries retry ≤ 3 and never
on 401/403, mutations `retry: false`, `focusManager` ↔ AppState, `onlineManager` ↔ `expo-network`.
Data changes because *we* invalidate it; time-based data overrides `staleTime` per query with a comment
(`staleTime: 5 * 60_000, // trends move during the day`).
Screens that must refresh on revisit use `useInvalidateOnFocus({ queryKey })`; main lists get pull to refresh.

## Mutations
```ts
// api/hooks/routine/useDeleteRoutine.ts
export const useDeleteRoutine = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (routineId: string) => deleteRoutine(routineId),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: queryKeys.routine._def });
      // the backend also drops the routine from the plan
      queryClient.invalidateQueries({ queryKey: queryKeys.plan._def });
    },
  });
};
```
* Invalidate the domain with `._def`; cross-domain keys only for backend side effects, each with a why-comment.
* **Mutations never retry.** The caller awaits the outcome before success feedback
  (`mutate(vars, { onSuccess, onError })` or `await mutateAsync`). Logout / delete account → `queryClient.clear()`.

**Optimistic update + rollback** - toggles, removes, likes (instant UI, cheap to undo):
```ts
onMutate: async (suggestionId) => {
  await queryClient.cancelQueries(queryKeys.suggestion.mine);
  const previous = queryClient.getQueryData<Suggestion[]>(queryKeys.suggestion.mine.queryKey);
  queryClient.setQueryData<Suggestion[]>(queryKeys.suggestion.mine.queryKey,
    (old) => old?.filter((s) => s.id !== suggestionId));
  return { previous };
},
onError: (_error, _vars, context) => {
  if (context?.previous) queryClient.setQueryData(queryKeys.suggestion.mine.queryKey, context.previous);
},
onSettled: () => queryClient.invalidateQueries(queryKeys.suggestion.mine),
```

**Device / SDK calls are mutations** (permission requests, purchases, share, capture):
`useMutation({ mutationFn: (pkg: PurchasesPackage) => Purchases.purchasePackage(pkg), networkMode: 'always', retry: false })`.
`networkMode: 'always'` because it is not a network request in TanStack's sense. No hand-rolled loading
flags, no refs guarding double taps (`isPending` disables the button). A user cancellation is not an
error (`isUserCancellation(error)` → do nothing).

## Errors
The backend answers `{ error, code }`. Map codes to i18n keys in one place:
```ts
// api/lib/errors.ts
const ERROR_MESSAGE_KEYS: Record<string, TranslationKey> = {
  ROUTINE_LIMIT_REACHED: 'errors.routineLimitReached',
  NAME_TAKEN: 'errors.nameTaken',
};
export const getErrorMessageKey = (error: unknown): TranslationKey => {
  if (isNetworkError(error)) return 'errors.network';
  const code = getApiErrorCode(error);
  return (code && ERROR_MESSAGE_KEYS[code]) || 'errors.generic';
};
```
Global codes are handled once in the axios response interceptor (`NOT_PRO` → paywall route; 401 →
single-flight token refresh, then logout) - never per screen. A Zod parse failure is a bug: it surfaces
as the generic error and is logged in dev. Queries: the screen draws the error state with retry.
Mutations: the feature hook toasts the mapped message.
