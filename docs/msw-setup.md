# MSW Setup in Olo Mobile

**Mock Service Worker (MSW)** intercepts network requests (GraphQL queries, REST calls) and returns fake responses. This means our tests never hit a real server — they run fast, offline, and are fully predictable.

> The MSW files live in:
> `applications/olo.native/src/__fixtures__/msw/`

---

## Folder structure

```
msw/
├── handlers.ts              ← Defines WHAT to intercept and HOW to respond
├── server.ts                ← Starts the mock server for Jest tests (Node.js)
├── server-native.ts         ← Starts the mock server for React Native runtime
└── msw-mock-responses/      ← Fake data returned by each handler
    ├── menu-query.ts
    ├── menu-page-headers.ts
    ├── success-menu-page-headers.ts
    ├── offers-query.ts
    ├── portion-menu-item-query.ts
    ├── pre-order-estimate-query.ts
    ├── saved-vouchers-query.ts
    ├── send-fos-event.ts
    ├── store-query.ts
    ├── store-search-query.ts
    ├── timed-order-slots-query.ts
    └── validate-basket.ts
```

---

## 1. Mock responses (`msw-mock-responses/`)

Each file exports one or more JavaScript objects that mirror the shape of a real GraphQL API response. These are the "fake answers" the mock server will return.

For example, `validate-basket.ts` looks like:

```ts
export const successValidateBasket = {
  data: {
    validateBasket: {
      basket: { id: '...', lines: [], total: 0, /* ... */ },
      validationErrors: [],
    },
  },
}
```

Some files export multiple variants to simulate different scenarios (e.g. a valid store vs. an invalid store). For example, `store-query.ts` exports `invalidStoreQuery`, `darlinghurstStoreQuery`, and `hamiltonStoreQuery`.

---

## 2. Request handlers (`handlers.ts`)

`handlers.ts` is the central file that tells MSW: *"When a request matching this name comes in, respond with this data."*

It imports mock responses and maps them to GraphQL operations:

```ts
import { graphql, rest } from 'msw'
import { successMenu } from './msw-mock-responses/menu-query'

export const handlers = [
  // When the app sends a "menu" GraphQL query, return fake menu data
  graphql.query('menu', (req, res, ctx) =>
    res(ctx.data(successMenu.data))
  ),

  // Some handlers branch based on input variables
  graphql.query('storeQuery', (req, res, ctx) => {
    if (req.variables.storeNo === 1111) {
      return res(ctx.data(invalidStoreQuery.data))   // simulate an error
    }
    return res(ctx.data(hamiltonStoreQuery.data))     // happy path
  }),

  // REST endpoints are also intercepted (e.g. analytics we don't need)
  rest.post('*/symbolicate', (req, res, ctx) =>
    res(ctx.status(403))
  ),
]
```

**In short:** handlers define *what* gets intercepted and *which* mock response to return.

---

## 3. Server setup (`server.ts` / `server-native.ts`)

These files create the actual mock server instance by passing the handlers in.

**For Jest tests** — `server.ts`:

```ts
import { setupServer } from 'msw/node'
import { handlers } from './handlers'

export const server = setupServer(...handlers)
```

**For React Native runtime** — `server-native.ts`:

```ts
import 'react-native-url-polyfill/auto'
import { setupServer } from 'msw/native'
import { handlers } from './handlers'

export const server = setupServer(...handlers)
```

The only difference is the import source (`msw/node` vs `msw/native`) and the URL polyfill needed by React Native.

---

## 4. Test lifecycle integration

The server is wired into every test automatically via `tools/setup-tests.js`:

```js
// Start intercepting requests before any test runs
beforeAll(() => server.listen({ onUnhandledRequest: 'warn' }))

// Reset handlers after each test so per-test overrides don't leak
afterEach(() => server.resetHandlers())

// Stop the server after all tests finish
afterAll(() => server.close())
```

This means **every test in the project** automatically gets mocked API responses without any extra setup.

---

## 5. How to add a new mock

**Step 1.** Create a response file in `msw-mock-responses/`:

```ts
// msw-mock-responses/my-new-query.ts
export const successMyNewQuery = {
  data: {
    myNewQuery: { /* shape matching your GraphQL schema */ },
  },
}
```

**Step 2.** Add a handler in `handlers.ts`:

```ts
import { successMyNewQuery } from './msw-mock-responses/my-new-query'

export const handlers = [
  // ... existing handlers
  graphql.query('myNewQuery', (req, res, ctx) =>
    res(ctx.data(successMyNewQuery.data))
  ),
]
```

**Step 3.** (Optional) Override a handler in a specific test:

```ts
import { graphql } from 'msw'
import { server } from '../__fixtures__/msw/server'

test('handles error case', () => {
  server.use(
    graphql.query('myNewQuery', (req, res, ctx) =>
      res(ctx.errors([{ message: 'Something went wrong' }]))
    ),
  )
  // ... test code
})
```

The override only lasts for that test — `afterEach(() => server.resetHandlers())` clears it automatically.

---

> **Note:** The project currently uses **MSW v1.3.1** which uses the `(req, res, ctx) =>` callback style. The latest MSW v2 uses a different syntax with `HttpResponse.json()`. If/when upgrading, the handlers will need to be migrated — see the [MSW v2 migration guide](https://mswjs.io/docs/migrations/1.x-to-2.x).

## 6. MSW v1 vs v2 — What changes if you upgrade?

MSW v2 introduced **breaking changes** to the handler syntax. It is not a drop-in upgrade — every handler in `handlers.ts` will need to be manually rewritten.

**v1 (current):**

```ts
graphql.query('myNewQuery', (req, res, ctx) =>
  res(ctx.data(successMyNewQuery.data))
)
```

**v2 equivalent:**

```ts
graphql.query('myNewQuery', () =>
  HttpResponse.json({ data: successMyNewQuery.data })
)
```

Key differences in v2:

| v1 | v2 |
|---|---|
| `(req, res, ctx) =>` callback | Simple resolver function, no `res`/`ctx` |
| `res(ctx.data(...))` | `HttpResponse.json(...)` |
| `res(ctx.errors(...))` | `HttpResponse.json({ errors: [...] })` |
| `res(ctx.status(403))` | `new HttpResponse(null, { status: 403 })` |
| `import { rest } from 'msw'` | `import { http } from 'msw'` (`rest` is renamed) |

The [MSW v2 migration guide](https://mswjs.io/docs/migrations/1.x-to-2.x) walks through all the changes in detail.
