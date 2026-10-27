---
id: coding-standards-vuejs
scope: scoped
severity: warning
audience: all
applies-to:
  - "**/*.vue"
  - "**/*.ts"
schemaVersion: 1
---

# Coding standards: Vue.js

The Vue.js and TypeScript specifics on top of the coding standards for all languages (`coding-standards-all-languages`), which are not repeated here. Both always apply together.

A `TODO(team)` marker is an open project decision. An AI assistant never fills one in itself. It asks.

## Most-violated rules

The rules below are the ones most often broken. Each is explained in its section.

1. `<script setup lang="ts">` with type-based `defineProps` and `defineEmits`. No Options API.
2. Use `@pareto4/ui-vue` where it has a component, and read its props from its type declarations. Never guess a prop.
3. No new dependencies without an explicit request.
4. Server data is a `RequestState<T>`. Data-loading composables and store actions never reject, and components never `try`/`catch`.
5. Nothing calls `fetch` except the HTTP core. Every request gets an `AbortSignal` from its caller.
6. View state lives in the URL. Screens get params and query as props, never through `useRoute()`.
7. State lives in the smallest place that works. A Pinia store only for state shared across screens.
8. Derived state is a `computed`. No watcher that keeps one ref in sync with another.
9. `v-for` keys are stable domain IDs. User-facing text comes from vue-i18n.
10. No `any`, no `!`, no casts, except the one typed response cast in API clients.
11. Work is done when the [definition of done](#definition-of-done) passes.

## Basic Stack

- Vue 3 (latest minor, 3.5 or higher) with the Composition API
- TypeScript in strict mode, type-checked with `vue-tsc`
- Vite
- Pinia, only when the app has state shared across screens (see [State management](#state-management))
- Vue Router
- vue-i18n
- `@pareto4/ui-vue` design system, styled with Tailwind CSS v4
- Vitest, Vue Test Utils and happy-dom (jsdom for axe checks)
- MSW for mocking the network in tests
- Playwright
- ESLint (`eslint-plugin-vue`, `typescript-eslint`) and Prettier
- Node latest LTS, pinned in `.nvmrc` and the `engines` field

The app is a browser-only single-page app. No SSR, no Nuxt.

## Dependencies

- **No new dependency without an explicit request.** This applies to runtime and dev dependencies alike. An AI assistant proposes a dependency and waits. It never installs one on its own initiative.
- **The stack and the platform come first.** Before proposing a package, check whether the stack, `@pareto4/ui-vue` or the platform covers it: `Intl`, `AbortSignal.any`, `structuredClone`, `Array.prototype.toSorted`.
- **No date, utility, validation or form libraries.** `Intl` formats, plain TypeScript does the rest. Examples of what is not added: lodash, dayjs, date-fns, zod, vee-validate.

## File and project organization

- **Feature folders.** Code lives in `src/features/<feature>/`. The feature folder is the frontend's version of the aggregate folder.
- **The folder layout is fixed:**

  ```
  src/
    main.ts
    App.vue
    style.css
    router/
      router.ts                    # createRouter, the global guard, onError
      route-names.ts               # RouteName
    shared/
      api/
        http.ts                    # the HTTP core
        api-error.ts               # ApiError, ApiErrorCode, toApiError
      state/
        request-state.ts           # RequestState<T>
      time/
        clock.ts                   # clock
      utils/
        assert-never.ts            # assertNever
    features/
      customer/
        index.ts                   # the feature's public API
        customer-routes.ts         # customerRoutes, added to the route table in router.ts
        customer-models.ts         # request and response interfaces, shared types
        customer-list-query.ts     # parseCustomerListQuery
        customer-test-ids.ts       # data-testid constants
        api/customer-api.ts
        composables/use-customer.ts
        composables/use-customer-search.ts
        components/CustomerCard.vue
        screens/CustomerListScreen.vue
        stores/                    # only when the feature has shared state
      session/
        stores/session-store.ts
  tests/
    helpers/with-setup.ts
    msw/                           # MSW server and handlers
    features/customer/...          # mirrors src/
  e2e/
  ```

- **Routed screens live in `screens/`** and their names end in `Screen` (`CustomerDetailScreen.vue`).
- **A feature's public API is its `index.ts`.** Another feature imports only from `@/features/<feature>`, never from a file inside it. Files inside a feature never import from their own `index.ts`.
- **Feature-internal components are imported only by their own feature**, and are never exported from `index.ts`.
- **`src/shared/` is a privilege, not a default.** Code moves there only when more than one feature uses it. A component more than one app needs belongs in `@pareto4/ui-vue`.
- **One component per `.vue` file**, named in PascalCase after the component (`CustomerListTile.vue`).
- **Other files are kebab-case** and named after what they export: `session-store.ts` exports `useSessionStore`, `use-customer-search.ts` exports `useCustomerSearch`, `customer-api.ts` exports `customerApi`. `index.ts` is the only exception.
- **Types shared between files get their own file** (`customer-models.ts`). A type used by one file stays in that file.
- **A `.vue` file exports only its component.** Types and helpers it shares with other files live in a `.ts` file.
- **Imports use the `@/` alias** for anything outside the current folder. No `../../` chains.
- **Named exports only.** Default exports exist only where a tool requires them: `.vue` files and config files (`vite.config.ts`).
- **`main.ts` wires the app in a fixed order.** Pinia, when the app uses it, before the router (guards use stores), then `app.config.errorHandler`, then the `vite:preloadError` listener, then `mount()` last.
- **`App.vue` holds the shell only.** Layout, the router view and app-wide hosts (`UiDialogHost`). No feature code.

## Shared building blocks

These types and modules are used everywhere. Use them exactly as defined here. Do not write variants.

```ts
// src/shared/state/request-state.ts
import type { ApiError } from '@/shared/api/api-error'

export type RequestState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: ApiError }
```

```ts
// src/shared/api/api-error.ts
// These codes are client categories, mapped from the HTTP status. The API's own `code` is in ApiError.problemCode.
export type ApiErrorCode =
  | 'aborted' // the caller aborted the request; not an error state
  | 'network' // no response
  | 'timeout'
  | 'unauthorized' // 401
  | 'forbidden' // 403
  | 'not-found' // 404
  | 'conflict' // 409
  | 'rate-limited' // 429
  | 'validation' // 400 or 422 with field errors
  | 'server' // 5xx
  | 'unexpected' // anything else

export interface ApiError {
  code: ApiErrorCode // the client category; components branch on this
  problemCode: string | null // the API's `code` ('CUSTOMER_NOT_FOUND'), null without a problem body; for the HTTP core and a feature that must tell two API errors apart
  status: number | null // null when there was no response
  traceId: string | null
  validationErrors: Readonly<Record<string, readonly string[]>> // i18n codes per request member; empty unless 'validation'
}

// The HTTP core rejects only with ApiRequestError. toApiError(e: unknown): ApiError
// turns anything caught into an ApiError, with code 'unexpected' for unknown values.
```

```ts
// src/shared/time/clock.ts
export const clock = {
  now: (): Date => new Date(),
}
```

```ts
// src/shared/utils/assert-never.ts
export function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${String(value)}`)
}
```

The HTTP core (`src/shared/api/http.ts`) exports `http` with this shape:

```ts
export interface RequestOptions {
  signal: AbortSignal
  idempotencyKey?: string
}

export interface HttpClient {
  get(path: string, options: RequestOptions): Promise<unknown>
  post(path: string, body: unknown, options: RequestOptions): Promise<unknown>
  put(path: string, body: unknown, options: RequestOptions): Promise<unknown>
  patch(path: string, body: unknown, options: RequestOptions): Promise<unknown>
  delete(path: string, options: RequestOptions): Promise<void>
}
```

The test helper `withSetup` lives in `tests/helpers/with-setup.ts`.

## Reference examples

These examples follow every rule in this document. New code follows their shape.

**Models and API client**

```ts
// src/features/customer/customer-models.ts
export interface CustomerResponse {
  id: string
  displayName: string
  email: string | null
  createdAt: string // ISO 8601, UTC
}
```

```ts
// src/features/customer/api/customer-api.ts
import type { CustomerResponse } from '@/features/customer/customer-models'
import { http } from '@/shared/api/http'

export const customerApi = {
  async getCustomer(id: string, signal: AbortSignal): Promise<CustomerResponse> {
    const body = await http.get(`/customers/${encodeURIComponent(id)}`, { signal })
    return body as CustomerResponse
  },
}
```

**Composable that loads data**

Server data that one screen uses is loaded by a composable, not a store.

```ts
// src/features/customer/composables/use-customer.ts
import { onWatcherCleanup, readonly, ref, toValue, watch } from 'vue'
import type { MaybeRefOrGetter } from 'vue'
import { customerApi } from '@/features/customer/api/customer-api'
import type { CustomerResponse } from '@/features/customer/customer-models'
import { toApiError } from '@/shared/api/api-error'
import type { RequestState } from '@/shared/state/request-state'

export function useCustomer(id: MaybeRefOrGetter<string>) {
  const state = ref<RequestState<CustomerResponse>>({ status: 'idle' })

  watch(
    () => toValue(id),
    async (customerId) => {
      const controller = new AbortController()
      onWatcherCleanup(() => controller.abort())
      state.value = { status: 'loading' }
      try {
        const data = await customerApi.getCustomer(customerId, controller.signal)
        state.value = { status: 'success', data }
      } catch (e) {
        const error = toApiError(e)
        if (error.code === 'aborted') {
          return
        }
        state.value = { status: 'error', error }
      }
    },
    { immediate: true },
  )

  return { state: readonly(state) }
}
```

**Store**

A store exists because more than one screen needs the same state. The reason is written above it.

```ts
// src/features/session/stores/session-store.ts
import { defineStore } from 'pinia'
import { ref } from 'vue'
import { sessionApi } from '@/features/session/api/session-api'
import type { CurrentUserResponse } from '@/features/session/session-models'
import { toApiError } from '@/shared/api/api-error'
import type { RequestState } from '@/shared/state/request-state'

// Shared because the global route guard and several screens read the signed-in user.
// Cache invalidation: loaded once after sign-in, cleared by reset() on sign-out.
export const useSessionStore = defineStore('session', () => {
  const currentUser = ref<RequestState<CurrentUserResponse>>({ status: 'idle' })

  async function loadCurrentUser(signal: AbortSignal): Promise<void> {
    currentUser.value = { status: 'loading' }
    try {
      const data = await sessionApi.getCurrentUser(signal)
      currentUser.value = { status: 'success', data }
    } catch (e) {
      const error = toApiError(e)
      if (error.code === 'aborted') {
        currentUser.value = { status: 'idle' }
        return
      }
      currentUser.value = { status: 'error', error }
    }
  }

  function reset(): void {
    currentUser.value = { status: 'idle' }
  }

  return { currentUser, loadCurrentUser, reset }
})
```

**Routes and query parameters**

```ts
// src/features/customer/customer-list-query.ts
import type { LocationQuery } from 'vue-router'

export interface CustomerListQuery {
  search: string
  page: number
}

export function parseCustomerListQuery(query: LocationQuery): CustomerListQuery {
  const search = typeof query.search === 'string' ? query.search : ''
  const page = typeof query.page === 'string' ? Number.parseInt(query.page, 10) : 1
  return { search, page: Number.isInteger(page) && page > 0 ? page : 1 }
}
```

```ts
// src/features/customer/customer-routes.ts
import type { RouteRecordRaw } from 'vue-router'
import { parseCustomerListQuery } from '@/features/customer/customer-list-query'
import { RouteName } from '@/router/route-names'

export const customerRoutes: RouteRecordRaw[] = [
  {
    path: '/customers',
    name: RouteName.customerList,
    component: () => import('@/features/customer/screens/CustomerListScreen.vue'),
    props: (route) => parseCustomerListQuery(route.query),
  },
  {
    path: '/customers/:id',
    name: RouteName.customerDetail,
    component: () => import('@/features/customer/screens/CustomerDetailScreen.vue'),
    props: true,
  },
]
```

**Screen**

```vue
<!-- src/features/customer/screens/CustomerDetailScreen.vue -->
<script setup lang="ts">
import { useI18n } from 'vue-i18n'
import CustomerProfile from '@/features/customer/components/CustomerProfile.vue'
import { useCustomer } from '@/features/customer/composables/use-customer'

const { id } = defineProps<{ id: string }>()

const { t } = useI18n()
const { state } = useCustomer(() => id)
</script>

<template>
  <p v-if="state.status === 'loading'">{{ t('common.loading') }}</p>
  <p v-else-if="state.status === 'error'" role="alert">{{ t(`errors.${state.error.code}`) }}</p>
  <CustomerProfile
    v-else-if="state.status === 'success'"
    :display-name="state.data.displayName"
    :email="state.data.email"
  />
</template>
```

**Composable**

```ts
// src/features/customer/composables/use-debounced-text.ts
import { onScopeDispose, readonly, ref, toValue, watch } from 'vue'
import type { MaybeRefOrGetter } from 'vue'

export function useDebouncedText(source: MaybeRefOrGetter<string>, delayMs: number) {
  const value = ref(toValue(source))
  let timer: ReturnType<typeof setTimeout> | undefined

  watch(
    () => toValue(source),
    (next) => {
      clearTimeout(timer)
      timer = setTimeout(() => {
        value.value = next
      }, delayMs)
    },
  )

  onScopeDispose(() => clearTimeout(timer))

  return { value: readonly(value) }
}
```

## Components

- **`<script setup lang="ts">` in every component.** The Options API, `defineComponent`, mixins, render functions and JSX/TSX are banned.
- **Block order is `<script setup>`, `<template>`, `<style>`.**
- **Script order follows the flow.** Imports, props/emits/model, stores and composables, local state, computed, watchers, functions, lifecycle hooks.
- **Component names are multi-word** (`CustomerCard`, never `Card`), so they never collide with HTML elements.
- **The `Ui` prefix belongs to the design system.** An app component never uses it, so `UiCard` always means the kit's.
- **Components are imported where they are used.** App components are never registered globally.
- **Use the design system.** Where `@pareto4/ui-vue` has a component, the app uses it. No hand-written buttons, inputs, tables or dialogs, no copied kit source, and no second component library.
- **Read the kit's API, never guess it.** Before using a kit component, read its props, events and slots from the package's type declarations. A prop name or value that is not in the types does not exist. The package ships its own documentation: `COMPONENTS.md` (each component's purpose, props, slots and an example), `README.md`, and `ai/components.json` for tools. When they differ from the type declarations, the types win.
- **Presentational by default.** Props in, events out. Only routed screen components read stores, call actions and use data-loading composables, and they pass plain props down. A presentational component never imports a store.
- **Pass the smallest input.** A component that renders a name takes `name: string`, not the whole `Customer`.
- **Extract a component** when the same shape appears twice, the template has three or more distinct sections, or it passes 100 lines. A render function or a `computed` that returns VNodes is not a component.
- **No wrapper `div` by habit.** Vue 3 supports multiple root nodes. An element exists for layout or semantics. A component that receives attributes from its parent (`class`, `data-testid`) has a single root, or binds `$attrs` explicitly. With several roots, attributes do not fall through and Vue warns.
- **The DOM is touched only from `onMounted` on**, and only through template refs.
- **A component stops what it starts.** Listeners, intervals and subscriptions end in `onUnmounted`, or live in a composable that cleans up for it.

## Props, events and v-model

- **Props and events are declared by type.** `defineProps<{ name: string }>()` and `defineEmits<{ select: [id: string] }>()`. Runtime declarations (`defineProps(['name'])`, the object syntax) are banned.
- **Every emitted event is declared.** A re-emitted native event that is not declared fires twice, because the listener also falls through to the root element.
- **Defaults come from reactive props destructure** (`const { size = 'md' } = defineProps<...>()`). A destructured prop passed to `watch` or a composable is wrapped in a getter (`() => size`), or it loses reactivity.
- **Props are never mutated**, including the nested properties of an object prop. The owner changes its state after an event.
- **Events name domain facts** (`select`, `archive`), not DOM events. Their payload is data, never a DOM event or a component instance.
- **Casing.** Props and events are camelCase in script and kebab-case in parent templates (`:display-name`, `@item-selected`).
- **`defineModel` for two-way binding.** No hand-written `modelValue` prop plus `update:modelValue` emit. Anything richer than a form-like value stays with the component that owns it and changes through events.
- **A `defineModel` object is replaced, never mutated** (`model.value = { ...model.value, name }`). Mutating a property emits nothing, and the parent never hears of it.
- **A boolean prop describes state** (`disabled`, `loading`), never a mode. Two modes are two components, or a union-typed `variant` prop.
- **Slots are typed** with `defineSlots`.
- **A wrapper component controls its attributes.** It sets `defineOptions({ inheritAttrs: false })` and binds `$attrs` to the element that should receive them.
- **`defineExpose` is rare.** A parent that calls child methods through a template ref is temporal coupling. Use props and events. The exception is an imperative DOM action like `focus()`.
- **`provide`/`inject` uses a typed `InjectionKey<T>`**, and only for true subtree context (a form group, a dialog host). It is not a way to skip props in a deep tree.
- **Provided state is `readonly`**, with explicit functions to change it. A required injection is read through a composable (`useFormGroup()`) that throws when the provider is missing.

## Forms

- **No form library.** A form is a component with local refs or `defineModel`, built from kit inputs.
- **Business validation belongs to the API.** The frontend checks only what makes a request pointless, such as an empty required field. That check is a `computed` in the form. It disables the submit button, and the field shows the kit input's error state once it has been touched, never before.
- **Server validation errors arrive as `error.code === 'validation'`.** `error.validationErrors` holds i18n codes per request member. The HTTP core builds it from the API's `validationErrors` entries: the entry's `field` (a dotted path for nested members) is the key, and its `code` is the i18n code. The form shows each translated message at its field.
- **A submit button is disabled while its request is `loading`**, so a double click never sends twice.

## Reactivity

- **`ref` is the state primitive.** `reactive` is not used. It cannot be reassigned and it loses reactivity when destructured.
- **`shallowRef` for large or external data** that is replaced as a whole: API result lists, chart data, third-party class instances. It is updated by replacing `.value`, never by mutating its content.
- **Derived state is a `computed`.** A `watch` that writes another ref to keep it in sync is banned.
- **A `computed` is pure.** No side effects, no async work, no mutation of other state.
- **A `computed` result is read-only.** Sorting uses `toSorted()`, never an in-place `sort()` or `reverse()` on a reactive or computed array.
- **`watch` is for side effects only**, with an explicit source. It is preferred over `watchEffect`, whose dependencies are hidden.
- **No `deep: true` by default.** Watch the specific getter that matters (`() => filter.value.status`).
- **`immediate: true`** instead of calling the same function from `onMounted` and from a watcher.
- **Async work started by a watcher is cancelled** with `onWatcherCleanup` when the source changes again, so a slow old response never overwrites a newer one. `onWatcherCleanup` is called synchronously, before the first `await` in the callback. The cleanup also runs when the component unmounts, because its watchers stop then.
- **A watcher that reads the DOM uses `flush: 'post'`.** `flush: 'sync'` is not used.
- **`nextTick` and `setTimeout` are not timing fixes.** Using them to make an ordering problem go away is temporal coupling. Fix the data flow.
- **Template refs use `useTemplateRef`.** DOM elements and component instances are never stored in reactive state, or passed into stores and services.
- **Compare by ID, not by object identity.** A reactive proxy is never `===` to its raw object.

## Templates

- **`v-for` keys are stable domain IDs.** Index keys and object keys are banned.
- **`v-if` and `v-for` never share an element.** Filter the list in a `computed`.
- **`v-if` by default.** `v-show` only for elements that toggle often.
- **Template expressions stay simple.** A property, a single comparison or one call. No `filter`, `sort` or `map` in a template. Anything more becomes a named `computed`.
- **Directive shorthands always**: `:`, `@` and `#`.
- **Components are PascalCase and self-closing in templates** (`<CustomerCard />`).
- **Interactive elements are real elements.** `button` for actions, `RouterLink` or `a` for navigation. No click handlers on a `div` or `span`.
- **User-facing text comes from vue-i18n** (`t('customers.search')`), also in an app with one language. This includes `aria-label`, `title` and `placeholder`. No hard-coded strings in templates.
- **No inline `style` attribute**, except for a value computed at runtime (a progress bar width).

## Composables

- **A composable is a `use` function** in its own `use-<name>.ts` file, and creates fresh state per caller. State that must be shared is a store, never module-level state in a composable.
- **A function without reactive state or lifecycle is a plain function**, not a composable. `formatPeriod()` lives in a `.ts` module, not in `useFormatters()`.
- **Called synchronously at the top of `<script setup>`** or of another composable. Never inside a callback or condition, and never after an `await`.
- **Returns a plain object of refs and functions**, so callers can destructure it. State the caller must not change is returned `readonly`.
- **Inputs are `MaybeRefOrGetter<T>`**, read with `toValue()` inside the getter or effect that uses them, so a caller can pass a value, a ref or a getter and changes are still tracked.
- **More than one optional parameter becomes an options object.**
- **No hidden side effects.** A composable does not write store state, and does not touch the DOM except through refs passed to it.
- **A composable that loads data exposes a `readonly` `RequestState`.** It owns the `AbortController`, aborts in `onWatcherCleanup`, converts failures with `toApiError`, ignores `aborted`, and never rejects. This is the default for server data that one screen uses.
- **A composable cleans up what it starts.** Timers, event listeners and in-flight requests stop in `onScopeDispose`.

## State management

### Where state lives

State lives in the smallest place that works. Take the first row that fits.

| State | Where it lives |
| --- | --- |
| Used by one component | A `ref` in that component |
| Passed between a parent and its children | Props down, events up, `defineModel` for a form value |
| Logic or server data that one screen uses | A composable, with fresh state per caller (`useCustomer(id)`) |
| Context for one subtree (a form group, a dialog host) | `provide`/`inject` |
| View state: filters, sorting, paging, the selected tab | The URL, as query parameters |
| Read or changed by more than one screen, or must survive navigation | A Pinia store |

- **Pinia is not the default.** A store is a shared singleton that lives as long as the app. It costs indirection, global state and test setup, so it needs a reason. Typical reasons: the signed-in user and permissions that the route guard reads, a selection or draft built up across several screens, server data that several screens show and that is cached on purpose.
- **Not a reason for a store:** "it is server data", "the screen is big", "the app might grow", or saving a few props between a parent and its children.
- **A small app has no Pinia.** An app without shared state does not install it. Pinia is added with the first store that is needed. Adding it is a dependency change: an AI assistant proposes it and waits.
- **Moving to a store later is cheap.** A data-loading composable and a store action have the same `RequestState` shape, so the composable's body moves into an action when a second screen needs the data.
- **Shared state is never module-level state.** An exported `ref` in a plain module is a store without devtools, reset or test isolation. Use a store.

### Pinia stores

These rules apply once the app uses Pinia.

- **Setup stores only.** Option stores are not used.
- **The reason for a store is written above it**, in the same comment as its cache invalidation.
- **One store per feature concern**, in `stores/<name>-store.ts`. Store IDs are unique across the app.
- **State changes happen inside actions only.** Components never assign to store state.
- **`storeToRefs` for state, direct calls for actions.** Destructuring state without `storeToRefs` breaks reactivity.
- **A setup store returns all of its state.** Pinia only tracks what is returned. Unreturned refs break devtools and `storeToRefs`.
- **`useXStore()` is called inside setup, a composable, an action or a guard**, never at module top level.
- **Server data is a `RequestState<T>` union** of `idle`, `loading`, `success` and `error`. No separate `isLoading` and `error` flags, and no `data` ref that outlives the error state.
- **Actions catch failures and convert them to the `error` state** with `toApiError`. A rejection never escapes into a template.
- **An aborted request is not an error.** When `error.code === 'aborted'`, the action resets that entry to `idle` and returns.
- **Actions never reject.** A caller that does not await an action starts it with `void`.
- **A store never creates an `AbortController`.** Actions that load data take a `signal` from their caller.
- **Cached server data has an explicit invalidation** (`reset()`, reload on route enter), decided when the cache is added and written as a comment above the store.
- **Parameterized data is a keyed `Map`** inside one store. Store definitions are never created dynamically.
- **Cross-store reads are allowed, cross-store writes are banned.**
- **Stores are UI state.** API clients, mappers and other non-UI code never read a store. They take their input as arguments.

## Routing

- **Route names are constants** (`RouteName.customer`). String paths appear only in the route table and in redirect handling.
- **Each feature exports its routes** (`customerRoutes` in `customer-routes.ts`). `src/router/router.ts` combines them into the route table.
- **The URL is the contract.** Every screen a user can reach has a URL that works cold, in a new tab. Filters, sorting and paging are query parameters. Path segments are lowercase kebab-case.
- **Screens are lazy-loaded** with `component: () => import(...)`, never with `defineAsyncComponent`.
- **Screens receive the URL as props.** Path params arrive with `props: true`. Query parameters arrive through a `props` function that parses and validates them (`parseCustomerListQuery`), so invalid input becomes a default, never an error. A screen never calls `useRoute()`.
- **A screen writes view state back with `router.replace({ name, query })`**, built from its own props.
- **A screen reacts to param changes.** Vue Router reuses the component when only the params change, so data is loaded by a watcher on the param prop with `immediate: true`, not in `onMounted`.
- **One global `beforeEach` guard handles authentication and authorization.** Route `meta` flags parameterize it. No per-screen checks, and no `beforeEnter` for access control, because it does not run when only the params change.
- **Guards return their result.** `false` cancels, a route location redirects. `next()` is banned.
- **`RouterLink` for anything that is a link.** `window.location` only for full-page exits (an external site, sign-out at the identity provider).
- **Dialogs are not routes**, unless they show a resource (a customer detail sheet). That is a child route.
- **A catch-all route renders a not-found screen.**
- **A lazy chunk that fails after a deploy reloads the page once.** A `vite:preloadError` listener in `main.ts` reloads, guarded by a `sessionStorage` flag. When the flag is already set, the error is not suppressed, `router.onError` reports it and the error screen shows, so a chunk that keeps failing never causes a reload loop.

## Screen patterns

The defaults for how screens are laid out and how they talk to the user. A project that needs something else records the decision.

- **Desktop first, never broken on a phone.** The primary target is an office laptop in a current Edge or Chrome. Layouts reflow to one column on narrow screens at Tailwind's default breakpoints, so nothing breaks on a phone. A phone-specific design is a project decision, and so are custom breakpoints.
- **The app shell is `UiAppShell` with `UiSidebar` as the only main navigation.** Items are grouped with the sidebar's `group` eyebrows, and the signed-in user and sign-out sit in its footer. There is no second navigation bar.
- **Navigation is at most two levels deep.** A detail screen has a back link to its list, and the list restores its filters from the URL.
- **No product tour and no help page by default.** Empty states guide first use.
- **An empty state is one sentence and the one action that resolves it**, shown with `UiEmptyState`: "No customers yet" with New customer, "No customers match these filters" with Clear filters. A list that is empty and a list emptied by its filters are different states with different actions.
- **Feedback appears where the action happened.** Field and form errors are inline at the field. A conflict or stale data is an inline alert on the screen with the action that resolves it (Reload).
- **Success is an inline status line** next to the action that caused it, with `role="status"` so it is announced, and it clears on the next action. The kit has no toast; a toast is proposed to `@pareto4/ui-vue`, never hand-written.
- **A destructive action is confirmed in a kit dialog** (`useDialog().confirm`) that names what will be lost.

## API access

- **One configured HTTP core** (`src/shared/api/http.ts`, a thin typed wrapper around `fetch`) holds the base URL, timeout, authentication, tracing and retry. Nothing else calls `fetch`. Axios is not used.
- **The HTTP core rejects only with `ApiRequestError`.** It maps the status code or failure to an `ApiErrorCode` and reads the `traceId` and the problem `code` (into `problemCode`) from the response.
- **One client module per backend resource** (`customer-api.ts`). It returns typed models, never a `Response` or raw JSON.
- **Request and response types are hand-written interfaces** in the feature's `<feature>-models.ts`, named after the backend DTO they mirror (`CustomerResponse`, `CreateCustomerRequest`). Required and nullable members follow the backend's OpenAPI document exactly. That document is served by the backend at `/v<n>/openapi.json` and exported with its build artifact.
- **Never guess a DTO member.** When the OpenAPI document is not available, an AI assistant asks instead of inventing fields.
- **A response is typed in one place.** The HTTP core returns `unknown`. An API client types it with `as <Dto>` directly on the result of an `http` call. This is the only cast allowed in the codebase: it trusts the OpenAPI contract, and there is no runtime validation.
- **API clients return the response interfaces as they are.** A mapper exists only when a screen needs a different shape. It is a plain function in the feature's `api/` folder.
- **Every request is cancellable.** The caller that starts the work owns the `AbortController`: a screen's param watcher or a composable. It passes the `signal` through the action to the API client, and aborts it in `onWatcherCleanup` or `onScopeDispose`.
- **Every request has a timeout.** The HTTP core combines the caller's signal with a timeout signal (`AbortSignal.any`). A hung request is an error, not an endless spinner.
- **Retries live only in the HTTP core**, for idempotent methods and for a `POST` or `PATCH` with an `Idempotency-Key`. A 4xx response is never retried, except `409 IDEMPOTENCY_REQUEST_IN_FLIGHT` and `429`: the core retries them with the same key, at most twice, after the `Retry-After` delay. A request the core may not retry, or a 429 still limited after the retries, rejects as `rate-limited`.
- **Components never `try`/`catch` an API call**, and never branch on a status code or message text. They render the `RequestState` and branch on `error.code`.
- **No floating promises.** Every promise is awaited or explicitly handled (`@typescript-eslint/no-floating-promises`). A promise that is deliberately not awaited is marked with `void`. That is safe for actions, because they never reject. The lint rule does not check templates, so a template handler calls a named function, never an inline promise chain.
- **Top-level `await` in `<script setup>` is banned.** It makes the component async and requires `<Suspense>`. Data is loaded by an action or composable, and the template renders its `RequestState`.

## Errors and logging

- **No `console` in committed code.** ESLint `no-console` is an error.
- **Unhandled errors are reported once**, by `app.config.errorHandler` and `router.onError`, to the project's telemetry.
- **Failed requests are reported once, by the HTTP core.** It reports `server`, `timeout` and `network` errors, with the `traceId`. Actions that convert a failure to the `error` state do not report it again.
- **Components and stores do not log.**
- **The generic error screen shows the `traceId`**, so support can find the request.
- **Never log response bodies, tokens or form values.** The `traceId`, status code and internal IDs are enough.

## Time, money and numbers

- **Time comes from the clock.** Logic reads "now" from `clock.now()` (`@/shared/time/clock`), never from `Date.now()` or an argumentless `new Date()`. Parsing a value (`new Date(iso)`) is allowed anywhere.
- **The clock is a module, not a Vue `provide`**, so stores and API clients use it too. Tests control time with fake timers (`vi.useFakeTimers()`, `vi.setSystemTime()`).
- **Instants travel as ISO 8601 strings in UTC.** Conversion to the user's time zone happens only for display.
- **Money and other business calculations are never done in the frontend.** Totals, VAT and rounding come from the API. The frontend only formats.
- **Formatting uses `Intl`** (`Intl.NumberFormat`, `Intl.DateTimeFormat`) with the user's locale. Values sent to the API are never locale-formatted.

## Code Standards for TypeScript

- **`strict` is on in every project**, together with `noUncheckedIndexedAccess`.
- **Warnings are errors.** ESLint runs with `--max-warnings 0`, and `vue-tsc --noEmit` is part of the build script. A bundle that ships is a bundle that type-checked.
- **ESLint uses `eslint-plugin-vue` `flat/recommended` and `typescript-eslint` `strictTypeChecked`.** Prettier owns formatting, with `printWidth: 120`.
- **`any` is banned.** Data of unknown shape is `unknown` and narrowed. Casts that silence the compiler (`as any`, `as unknown as T`) are banned. The only allowed cast is the typed response in an API client.
- **The non-null assertion `!` is banned** outside test code. A template ref is typed `| null` and checked.
- **`@ts-ignore` is banned.** `@ts-expect-error` is allowed only with a reason and a TODO with a work item.
- **No TypeScript `enum`.** Use a string-literal union, or an `as const` object when the values are needed at runtime.
- **`interface` for object shapes, `type` for unions and aliases.**
- **`import type` for type-only imports**, enforced by `verbatimModuleSyntax`.
- **Exhaustive unions.** A `switch` over a union ends in a `default` that calls `assertNever(value)`, so a new member is a compile error.

## Enforcement

The rules below are enforced by lint or the compiler. Every other rule in this document is checked in review, and an AI assistant checks it itself before it reports work as done.

| Rule | Enforced by |
| --- | --- |
| Only `<script setup>` | `vue/component-api-style` (`['script-setup']`) |
| Type-based props and emits | `vue/define-props-declaration`, `vue/define-emits-declaration` |
| Block order and macro order | `vue/block-order`, `vue/define-macros-order` |
| Every emitted event declared | `vue/require-explicit-emits` |
| User-facing text from vue-i18n | `vue/no-bare-strings-in-template` |
| No static inline `style` | `vue/no-static-inline-styles` |
| No `v-html` | `vue/no-v-html` (off only in the sanitizing component) |
| Composables not after `await` | `vue/no-restricted-call-after-await` |
| No `reactive` | `no-restricted-imports` on `reactive` from `vue` |
| No `fetch` outside the HTTP core | `no-restricted-globals` on `fetch`, off in `http.ts` |
| No `Date.now()` or argumentless `new Date()` outside the clock | `no-restricted-syntax` with `CallExpression[callee.object.name='Date'][callee.property.name='now']` and `NewExpression[callee.name='Date'][arguments.length=0]`, off in `clock.ts` |
| No `console` | `no-console` |
| No floating promises | `@typescript-eslint/no-floating-promises` |
| Feature isolation, no store in presentational components | `eslint-plugin-boundaries` TODO(team): configure |
| Strict types, no unchecked index access | `vue-tsc --noEmit` |

## Styling

- **Tailwind utilities with the design system's semantic tokens.** No hard-coded colors, spacing or font sizes. They are magic values. The tokens are defined by the kit in `@pareto4/ui-vue/theme.css`.
- **Tailwind v4 is configured in CSS.** `src/style.css` imports Tailwind and the kit's tokens, and Vite uses `@tailwindcss/vite`. There is no `tailwind.config.js`.
- **Class names are written out in full.** Tailwind only generates classes it finds in the source, so a class built by concatenation (`` `bg-${tone}` ``) renders unstyled. Map a value to a complete class name instead.
- **`<style scoped>` only for what utilities cannot express**, with class selectors, not element selectors. Global CSS lives only in `src/style.css`.
- **No `:deep()` into design-system components.** A needed change is proposed to `@pareto4/ui-vue`.
- **Layout differences are viewport or container sizes, handled in CSS.** Never user-agent sniffing.
- **Animations use `transform` and `opacity` only.**

## Security

- **`v-html` is banned.** The only exception is one shared component that sanitizes its input with DOMPurify.
- **URLs from data bound to `href` or `src` are validated** to `https:` or a relative path. Vue does not block `javascript:` URLs.
- **Nothing secret in the bundle.** Every `VITE_` variable ships to the user. No API keys, tokens or connection strings.
- **No tokens or personal data in `localStorage` or in the URL.**

## Accessibility

There is no formal WCAG level. These rules are the minimum every screen meets.

- **Every input has a label** (the kit's `label` prop), and every icon-only button a `title` or `aria-label`.
- **Everything works with a keyboard.** Hover-only actions are banned.
- **Each screen has an axe check** in its component test (`vitest-axe`). vitest-axe does not work under happy-dom, so a screen's spec file starts with `// @vitest-environment jsdom`.
- **axe in a simulated DOM does not check color contrast.** Contrast is covered by the design system's tokens.

## Performance

- **Optimize when measured.** `v-memo`, `v-once` and hand-written memoization need a measured problem. An AI assistant never adds them unless asked.
- **Routes are code-split.** Heavy components that are not visible on first render use `defineAsyncComponent`, with an error component.
- **Lists that can grow without bound are paged or virtualized.** Never thousands of rendered rows.
- **List rows get precomputed props** (`:selected="item.id === selectedId"`), not the shared value, so one change does not re-render every row.
- **`KeepAlive` always has a `max`**, and never caches screens with sensitive data.
- **`onUpdated` does no API calls and no state changes.**

## Testing in Vue

- **Stores, composables and API clients follow the full testing rules.** They are the frontend's application and adapter code.
- **Components are tested on their contract.** Props and store state in, rendered output and emitted events out. Never assert on `wrapper.vm` or internal state.
- **Tests live in `tests/`, mirroring `src/`**, as `<name>.spec.ts` (`tests/features/customer/components/CustomerCard.spec.ts`). `src/` holds production code only.
- **Naming.** `describe` names the component or function, a nested `describe` the method or interaction, and `it` the state and expected behavior. Together they read as `MethodName_StateUnderTest_ExpectedBehavior`.
- **`mount`, not `shallowMount`.** Child components stay real.
- **Mock only the network, with MSW.** The HTTP core stays real in every test, so its error mapping is tested too. The MSW server and shared handlers live in `tests/msw/`. A test overrides a handler with `server.use()` for its own case.
- **Elements are found by `data-testid`**, from a per-feature constants file (`customer-test-ids.ts`), named `<feature>-<screen>-<element>`. Visible text is only used to assert content.
- **Interactions are awaited.** `await trigger()` and `await setValue()`, then `await flushPromises()` for mocked async work. A single `nextTick` is for a state change the test made itself.
- **A composable that uses lifecycle hooks or `inject` is tested inside a host component** (`withSetup` from `tests/helpers/with-setup.ts`). Other composables are called directly.
- **A test that touches a store gets a fresh Pinia.** `createTestingPinia` for component tests, `setActivePinia(createPinia())` for store tests.
- **Time is controlled with fake timers** (`vi.useFakeTimers()`), restored after each test.
- **Snapshots only for small leaf components**, one per visual state, reviewed like code, and never the only assertion. A view is asserted by behavior.
- **E2E tests use Playwright** and live in `e2e/` next to `src/`, in the app's folder under `code/`. They navigate by URL and cover the critical user journeys only.
- **Coverage uses `@vitest/coverage-v8`.** Generated code and type-only files are excluded in `vitest.config.ts`.

## Definition of done

A change is done when all of these pass without errors or warnings. An AI assistant runs them itself and does not report work as done before they pass. Every app's `package.json` has these script names, so the commands are the same in every repository.

1. `npm run lint`
2. `npm run type-check` (runs `vue-tsc --noEmit`)
3. `npm run test` for the specs of every changed file, and the specs of the files that import it

A change that adds a user journey or changes one also runs the affected Playwright tests.