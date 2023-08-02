# Chapter 4

### Hooks

One of the last major concepts to understand in Feathers, Hooks are simply functions that can be configured to run at different points in time in a request's lifecycle. They are generally defined as being before, after, on an error, or around a service call. For example, there could be a `before` hook that authenticates a user before the rest of the query is executed, or there could be an `after` hook that triggers a notification to be send after the query is completed. 



### General structure of hooks

Hooks are generally defined in the main `.ts` file of a service (the file that does not have a specific suffix). In the example project so far, it would be `students.ts`

```ts
  app.service(studentPath).hooks({
    around: {
      all: [
        schemaHooks.resolveExternal(studentExternalResolver),
        schemaHooks.resolveResult(studentResolver)
      ]
    },
    before: {
      all: [
        schemaHooks.validateQuery(studentQueryValidator),
        schemaHooks.resolveQuery(studentQueryResolver)
      ],
      find: [],
      get: [],
      create: [
        schemaHooks.validateData(studentDataValidator),
        schemaHooks.resolveData(studentDataResolver)
      ],
      patch: [
        schemaHooks.validateData(studentPatchValidator),
        schemaHooks.resolveData(studentPatchResolver)
      ],
      remove: []
    },
    after: {
      all: []
    },
    error: {
      all: []
    }
  })
```

Notice a few things regarding the above structure:

1. For each type of hooks (before, after, error and around), we can define them in either all methods, or only in speific ones (find, get, create, patch and remove).
2. You will see exactly where validators and resolvers are called. For the `create` and `patch` hooks, you will also see the correct order - validator and then resolver.
3. Notice that the order is reversed for the `around` hooks calling the resolver, and then the validator. We will talk about why this matters for `around` hooks later on in the chapter.



### Using before, after, and error hooks

Hooks are actually just `async` functions, with a special `HookContext` argument passed to them. For example, we could define a very simple hook that simply logs that it is being run:

```ts
export const logHere = () => async (context: HookContext) => {
  console.log("The hook is running!")
  return context;
};
```

> There is no official file / location to put defined hooks like the above. Typically, I create a new file for each service, `.hooks.ts`, and place hooks related to that service there. If its a hook expected to be used accross multiple services, put them in the `src/hooks` folder.



We could then add the hook somewhere in the main service file:

```ts
create: [
  schemaHooks.validateData(studentDataValidator),
  schemaHooks.resolveData(studentDataResolver),
  logHere()
],
```



We could also pass data into the hook to customize their behaviour when being called from different places. For example:

```ts
export const logHere = (name: string) => async (context: HookContext) => {
  console.log(`The hook ${name} is running!``)
  return context;
};
```

```ts
create: [
  logHere('before validators and resolvers')
  schemaHooks.validateData(studentDataValidator),
  schemaHooks.resolveData(studentDataResolver),
  logHere('after validators and resolvers')
],
```

When a `create` request is received by the above service, it would print:

```
The hook before validators and resolvers is running
The hook after validators and resolvers is running
```



### The `context` object

The `context` object contains information pertaining to that current request. There are a [multitude of properties](https://feathersjs.com/api/hooks.html#hook-context) that you should familiarize yourself to, but some you ***need*** to take note are:

| Property                  | Description                                                  |
| ------------------------- | ------------------------------------------------------------ |
| `context.method`          | The method of the request, which could be `get`, `find`, `create`, `patch` or `remove`. |
| `context.id`              | In `get` , `patch` or `remove` request, this is the `id` of the resource request. For example in a traditional rest API call to `localhost:3000/users/5`, `id` would be `5`. |
| `context.data`            | The object containing the data for `create` and `patch` calls. |
| `context.result`          | The object to be returned to the client. This is usually populated during the service call, and is available to read only on `after` hooks. If this is set manually in a `before` hook, i.e. before the service call, the **service call will be skipped**. This is useful for mocking, or caching. |
| `context.params`          | Defined to contain "additional information" for the request, we can use this to store and read custom parameters throughout our request. We will talk more about this in the future. |
| `context.params.query`    | The object containing the query params of the request. Most typically used in `find`, but any request can pass in query params. |
| `context.params.user`     | In the default authentication system, this object is the authenticated user that performed the request. If the request was performed by an unauthenticated user, this will be `undefined`. |
| `context.params.provider` | This `string` describes whether the request is an internal request, or from an outside client. If it is from an outside client, the value of `provider` would be the transport used to make the request (i.e. `rest` or `socketio`). If it is an internal call, it would be `undefined`. In testing, the value `external` can/should be used to refer to an external call. |

##### A note about `context.data` and `context.result`

It is worth. noting that `data` and `result` could have more than one form in Feathers. 

In a `find`, the `context.result` object can either be non paginated (for example, `Student[]`), or paginated (for example, `{ data: Student[], total: number, limit: number, skip: number }`).

Meanwhile, in a `get`, (or any other method actually), the `context.result` object is typically just one of the resource (in this case, just a `Student`).

Similarly, `context.data` could be an array of data, because Feathers has an optional behaviour of passingn in an array of data to create /patch multiple objects. 

The important thing to note about this is that, modifying individual `data` and `result` fields in hooks is tedious, because you have to account for the different forms of the fields. If you want to do this, you should do these in **Resolvers** instead.

> The `context` object is also accessible in each resolver function!



### Using `around` Hooks

`before`, `after`, and `error` hooks are pretty easy to wrap your head around - they are run one after the other, in the order they are defined in the main service file. 

`around` hooks are a bit more complicated, but allows us to do slightly more with them. Specifically, they are useful when we want to have more control over the whole request lifecycle in one hook. Let's take a look at a simple `around` hook:

```ts
export const logError = () => async (context: HookContext, next: NextFunction) => {
  try {
    await next()
  } catch (error: any) {
    logger.error(error.stack)

    // Log validation errors
    if (error.data) {
      logger.error('Data: %O', error.data)
    }

    throw error
  }
}

```

Notice that the function definition is slightly different from the other hooks - namely the `next` function. Instead of moving on to the subsequent hook when the hook returns, `around ` hooks run subsequent hooks (or even the service call itself) by calling `await next()`. This allows us to do a few neat things:

1. Wrap subsequent calls in a `try`, as shown above
2. Run code before the subsequent calls
3. Run code after the subsequent calls



Another simple use case for `around` hooks is a hook to time how long it takes for the rest of the call to run:

```ts
export const timer = () => async (context: HookContext, next: NextFunction) => {
  const startTime = Date.now()
  
  await next()
  
	const timeElapsed = Date.now() - startTime
  console.log(`It took ${timeElapsed}ms to run the rest of the request!`)
}

```



### Application-level Hooks

If you look at the `app.ts` file, you can actually see another set of application-level hooks. These are run for requests accross all services, although you could still define specific situations when the hooks should be skipped. There are also special `setup` and `teardown` hooks, which you can read more about [here](https://feathersjs.com/api/services.html#setup-app-path). 

```ts
app.hooks({
  around: {
    all: [logError]
  },
  before: {},
  after: {},
  error: {}
})
// Register application setup and teardown hooks here
app.hooks({
  setup: [],
  teardown: []
})
```



### Extra: Hook Order

Now that you are introduced to Hooks, one consideration when using them is the order that hooks are run. I believe that hooks are generally run in this squence:

`App Around (before await next())`

`App Before`

`Service Around (before await next())`

`Service Before`

`The service call`

`Service After `

`Service Around (after await next())`

`App After`

`App Around (after await next())`

*editor's note: this is what I believe the correct sequence is, but I should really test this theory one of these days to make sure lmao*



### Extra: Hook Definitions

Generally, you will see two forms of hook definitions:

```ts
export const logHere = () => async (context: HookContext) => {
  console.log("The hook is running!")
  return context;
};
```

and 

```ts
export const logHere2 = async (context: HookContext) => {
  console.log("The hook is running!")
  return context;
};
```

The only difference in these forms is how they are registered:

```ts
create: [
  logHere()
  logHere2
],
```

Notice that one needs to be called (with the `()`) while the other does not. 

I generally prefer to standardize my hooks to use the first formk, since in that form we can actually pass arguments to the hook - but you will see both definitions used in the community, and even in the generated code itself (e.g., the default `logError` hook uses the second form).

At the end of the day, it does not really matter which form is used, although I personally prefer the former. It is just importantn to understand that both forms exists, and that you should register them appropriately. (Although Feathers is very good at screaming at you should you register a hook incorrectly.) 