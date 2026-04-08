# @unruly-software

- [TypeScript ecosystem](#typescript-ecosystem)
  - [Domain modelling](#domain-modelling)
  - [Error and absence handling](#error-and-absence-handling)
  - [API layer](#api-layer)
  - [Testing](#testing)
  - [Quick look](#quick-look)
- [Rationale](#rationale)

## Summary

Small, composable typescript libraries that solve common data modelling, API,
and testing problems in a way that's easy to adopt and hard to misuse. Built on
the principle of validating data once at the boundary and carrying proof of
that validation through the type system.


---

## TypeScript ecosystem

### Domain modelling ⭐⭐⭐

| Package | Description |
|---|---|
| [`@unruly-software/value-object`](https://github.com/unruly-software/value-object) | Zod-backed value objects, runtime validation, structural equality, `JSON.stringify` round-tripping, and real classes you can attach methods to. No decorators, no `reflect-metadata`. |
| [`@unruly-software/entity`](https://github.com/unruly-software/entity) | Event-driven domain entities on top of Zod. Define props and events once; get typed mutations, per-mutation rollback on validation failure, a domain event journal, and storage serialisation. |

### API layer ⭐⭐⭐

| Package | Description |
|---|---|
| [`@unruly-software/api`](https://github.com/unruly-software/api) | Define an API in Zod once, then drive your client, server, React Query hooks, and anything else that needs to know the shape of a request or response without coupling those layers to each other. Includes `api-client` (core), `api-server`, `api-query`, and an experimental Express adapter. |


### Testing ⭐⭐⭐

| Package | Description |
|---|---|
| [`@unruly-software/faux`](https://github.com/unruly-software/faux) | Deterministic fixture generation. Define model factories with dependency injection and isolated cursor management; the same seed always produces the same data. Bring your own test framework. |

### Error and absence handling ⭐⭐

| Package | Description |
|---|---|
| [`@unruly-software/result`](https://github.com/unruly-software/result) | `Result<T, E>` and `AsyncResult<T, E>` typed sync/async error handling without try/catch noise. Chain `.map()`, `.mapAsync()`, and `.tap()` seamlessly across sync and async boundaries.</br>We no longer believe monadic types are the be-all and end-all of types. Adding complex result types your colleagues have to learn may not be worth it. But they can be fun. |
| [`@unruly-software/optional`](https://github.com/unruly-software/optional) | Monadic `Optional<T>` for null-safe chaining. Converts seamlessly between async and sync with a similar interface to `Result`</br>This package was made long before typescript added null coalescing operators so unless you love extra types this may not be the best for you |


---

### Quick look

```ts
/**
 * value-object: parse once at the boundary, carry the proof through the type
 * system. Equality and serialisation for free. Data that is meant to carry
 * immutable value, not identity and mutation.
 */
class Email extends ValueObject.define({
  id: 'Email',
  schema: () => z.string().email(),
}) {
  get domain() { return this.props.split('@')[1] }
}

const email = Email.fromJSON('alice@example.com') // throws if invalid
email.domain // 'example.com' -- no re-validation needed downstream
JSON.stringify(email) // '"alice@example.com"' -- round-trips through JSON without losing the type

/**
 * entity: Event-driven domain entities with Zod validation. Define props and
 * events once; get typed mutations, per-mutation rollback on validation
 * failure, a domain event journal, and storage serialisation.
 *
 * For data that is meant to carry identity and mutation.
 */
class Account extends Entity.define(
  { name: 'account', idField: 'accountId', schema: () => z.object({ ... }) },
  [onCreated, onDeposited],
) {}

const account = new Account()
account.mutate('account.created', { name: 'Operating', tenantId: 'tenant-1' })
account.mutate('account.deposited', { amount: 250 })
account.props.balance // 250 -- schema re-validated after every mutation

/**
 * api: define your API in Zod once, then drive your client, server, React Query
 * hooks, and anything else that needs to know the shape of a request or
 * response -- without coupling those layers to each other.
 *
 * Use with value objects to have ergonomic, type-safe APIs with behaviour
 * attached to your data, without sacrificing type safety or runtime
 * validation.
 **/
const accountAPI = {
  getAccount: api.defineEndpoint({
    request: z.object({ accountId: z.number() }),
    response: z.object({ name: z.string() }),
    metadata: { method: 'GET', path: '/accounts/:accountId', somethingElse: '...' },
  }),
}

const router = defineRouter<typeof accountAPI, AppContext>({
  definitions: userAPI,
})
const implemented = router.implement({
  endpoints: {
    getAccount: router
      .endpoint('getAccount')
      .handle(({ context, data }) => context.accountService.findById(data.accountId)),
  },
})
implemented.dispatch({ endpoint: 'getAccount', data: { accountId: 123 } }) // typed request body, typed response body on the server

const client = new APIClient(accountAPI, { resolver: ...} // Typed API client with a customizable resolver

client.request('getAccount', { accountId: 123 }) // typed request body
  .then(account => account.balance) // typed response body




/**
 * faux: deterministic fixtures with complete intellisense. Define model
 * factories with dependency injection and isolated seed management; the same
 * seed always produces the same data. Bring your own test framework and make
 * testing with snapshots or making nonproduction data deterministic a breeze.
 *
 */
const context = faux.defineContext({
  helpers: {
    randomName: ({ seed }) => `User${seed}`,
    randomEmail: ({ seed }) => `user${seed}@example.com`
  }
  shared: () => ({
    timestamp: new Date('2024-01-01')
  })
})

const user = context.defineModel(ctx => ({
  id: ctx.seed,
  name: ctx.helpers.randomName,
  email: ctx.helpers.randomEmail
  createdAt: ctx.shared.timestamp
  // Resolve another model from a different file
  address: ctx.find(address)
}))

const fixtures = context.defineFixtures({ user, ... })

const f = fixtures({ seed: 123, override: { user: { name: 'John' }}})
f.user         // generated on demand, cached for this fixture instance
f.user.address // resolved from its own model, same seed offset every time

/**
 * result: typed sync/async error handling without try/catch noise. What I
 * hoped Promises would be when they were first released. Chain .map(),
 * .mapAsync(), and .tap() seamlessly across sync and async boundaries, with no
 * nesting or boilerplate.
 */
Result.invokeAsync(authorize, req)
  .mapAsync(getPostsForUser)
  .tap(
    posts => res.json({ posts }),
    err   => res.status(500).send('Something went wrong'),
  )

/**
 * optional: monadic Optional<T> for null-safe chaining. Converts seamlessly
 * between async and sync with a similar interface to Result. For cases where
 * the absence of a value is not an error, but you still want to avoid null
 * checks and `?.` noise.
 */
Optional.of(maybeUser)
   .pick('userId')
   .flatMapAsync(getUserSettings)
   .orElse(defaultSettings)
```


## Rationale

We've built and scaled software at companies of every size. Early startups
running on a single server, mid-stage teams navigating rapid growth, and large
organisations with complex distributed systems. Across all of them, the same
patterns kept proving their worth, and the same classes of problem kept causing
pain. These libraries are our attempt to package those patterns in a way that's
easy to adopt and hard to misuse.

Bad data leads to bad outcomes. A `string` that might be an invalid email, a
number that might be negative, an object that hasn't been validated since it
left the database. These are all correctness issues and have lead to more bugs,
edge cases and security vulnerabilities than we can count.

Our libraries are designed around the idea that data should be validated
exactly once, at the boundary where it enters your system, and carry proof of
that validation through the type system from that point on.

We also know that companies change shape. Code that runs in a single Lambda
today might need to run in a container tomorrow, or in a Kubernetes cluster
next year. Our packages have no opinion about your deployment runtime, your
HTTP framework, or your database. The business logic they help you model is the
part that should outlast those choices.
