# Chapter 3

### Resolvers

A resolver can be thought of a function that transforms data, at a per-field level of an object. In their simplests terms, you can think of them like this:

```Input Object -> Resolver -> Output Object```

The resolver takes an input object, acts on the individual fields of that object, and returns an output object. They can be used for a variety of things, such as:

1. Validation - e.g. when creating a new User, you can check if an email is already in-use by another account
2. Data Manipulation - e.g. when accepting an email property, you can ensure that the string given is in lowercase
3. Permission/role-based Manipulation - e.g. if the calling user is a non-admin, return `undefined` for certain fields
4. Field population - e.g. if the databse has a record for `userId`, a `user` object can be populated at query-time to fill in data related to that user that is useful for the request

> Many of these things could be done in Hooks as well, but we will discuss what actions are best done in Resolvers, and what are more appropriate for Hooks later on.



### Default Resolvers

Just like schemas, Feathers provides us with some default resolvers that act on data at different points in time to get us started.

| Resolver                | Purpose                                                      |
| ----------------------- | ------------------------------------------------------------ |
| studentResolver         | Resolving results of the raw service call                    |
| studentExternalResolver | Resolving resulst of the raw service call (specifically to external, client-side requests) |
| studentDataResolver     | Resolving `data` passed in through `create`                  |
| studentPatchResolver    | Resolving `data` passed in through `patch`                   |
| studentQueryResolver    | Resolving query params passed in through mainly `find`       |

> Remember that services can be called either internally by another service, or directly through the Feathers client. The "external" resolver only acts on non-internal requests.



To help understand what resolvers are run when, let's take a look at a couple of scenerios:



Example with a `find` request

1. When Feathers sees the `find` request, it validates the query parameters of the request using the `studentQueryValidator`, which is based on the `studentQuerySchema`.
2. The query params are then passed through the `studentQueryResolver`. The resolver performs validations and transforms the query params. Let's call this resolved query params `qp`
3. Feathers then uses `qp` to perform a `find` call on the database.
4. The data found is passed through the `studentResultResolver`, which data is then passed through the `studentExternalResolver`, since we are assuming this is a non-internal request.
5. The client receivese the final data.



Example with a `post` request

1. When Feathers sees the `create` request, it validates the `data` of the request using the `studentDataValidator`, which is based on the `studentDataSchema`.
2. The `data` then passes through the `studentDataResolver`. The resolver performs validations and transforms the `data`, lets call this resolved data `resolvedData`
3. The `resolvedData` is then passed into the service call. In this case, a database object is created with the data `resolvedData`. 
4. The database returns the newly-created object to Feathers
5. The data of the newly-created object is passed through the `studentResultResolver`, which data is then passed through the `studentExternalResolver`, since we are assuming this is a non-internal request.
6. The client receivese the data of the created object.



> If you have some understanding on how hooks work, you can take a look at the `student.ts` file to see exactly when different validators / resolvers are run. However, we will only be talking about Hooks in the next chapter, so don't worry about that too much!

Once again, this is merely what Feathers has set up for us by default. If we wanted, we could always add more or even choose to remove existing resolvers. Before doing that though, it might be worth taking a look at how they work.



### How the `resolve` function works

Let's take a look at the `studentResolver` function:

```ts
export const studentResolver = resolve<Student, HookContext>({})
```

The resolver takes in object (`{}` here) as an argument, but takes two *type parameters*, `Student` and `HookContext`. 

The type `Student` here determines what fields / properties the resolver will act on (regardless of whether they acually exist on the input data). For example, if you were to try adding properties onto the argument object, you would see that TypeScript prompts you to use one of the fields in the `Student` type. For example, the `name` field:

```ts
export const studentResolver = resolve<Student, HookContext>({
  name: async (value, instance, context) => value
})
```

> Note that technically, we could pass whatever we wanted instead of `Student`. We could pass in `StudentData`, or a combination of both using `Student & StudentData`, or whatever other type we wanted. However, by default, most resolvers have the main model schema as the first type parameter, except for the query resolver.



For each field (in this case `name`), you pass as `async` function that takes up to 3 parameters:

1. `value`: representing the value of that field in the input object
2. `instance`: representing the whole input object
3. `context`: representing the context of the request (typed as `HookContext`*)

In this async function, you are free to do anything you need to to the given value of the field. For example, since this is the `studentResolver`, this resolver takes in the raw database data as an input, and returns how the clients will see the data. For example, we could ensure that clients would only see the name in all caps:

```ts
export const studentResolver = resolve<Student, HookContext>({
  name: async (value, instance, context) => value.toUpperCase()
})
```

> It is worth noting that this will not do anything to the original input data, but only affects the returned, output data!



or, we could append an "(M)" or "(F)" at the of the name:

```ts
export const studentResolver = resolve<Student, HookContext>({
  name: async (value, instance, context) => value + ` (${instance.gender})`
})
```

or, we could hide the student's name if the user that requested the information is not an admin:

```ts
export const studentResolver = resolve<Student, HookContext>({
  name: async (value, instance, context) => {
  	const requestingUser = context.params.user
    if (requestingUser.role === 'admin') return value
    return undefined
  }
})
```

> Note that in this case, we are assuming that the User object has a `role` field. This is completely up to you, and the above is just for the sake of an example.

> *We will discuss HookContext in a future chapter. For now, just understand that it contains most of the relevant data for the current request. For example, `context.method` represents if it was a `create`, `patch`, `find`, `get` or `remove` request, while `context.params.query` represents the query params of the request, etc.



### A note about `instance`

```ts
export const studentResolver = resolve<Student, HookContext>({
  name: async (value, instance, context) => value
})
```

Note that in the async function above, I described `instance` as being the input object of the resolver. This is true, but the tricky part here is that the input object is actually different for different resolvers. For example, in the data and patch resolvers, it would be easier to think about the `instance` parameter to be the `data` passed to the request:

```ts
export const studentDataResolver = resolve<Student, HookContext>({
  name: async (value, data, context) => value
})
```

> For the sake of simplicity, I will continue to refer to the 2nd parameter of the async function as `instance`.



For the result resolver, `instance` is the database object's raw data, while in the query resoler, `instance` represents the possible query parameters, found in `context.params.query`. And lastly, the `instance` of the external resolver is actually the output of the result resolver.

These distinction are important to note so that you don't assume that data is populated when it is not. For example, let's say that when wanting to patch the `name` field of a piece of data, you want to append the gender to the name. The data sent would look like this :

```json
{
  "name": "Tommy"
}
```

And given a data resolver:

```ts
export const studentDataResolver = resolve<Student, HookContext>({
  name: async (value, data, context) => value + ` (${data.gender})`
})
```

You might assume that the name being saved to the databse would be `Tommy (M)`. However, it would actually be `Tommy ()`. This is because `data` in this case does not refer to the data of the database object, but refers to the data provided in the `patch` request (which does not container a `gender` property).

If you wanted to use the existing data of an object you would have to fetch it before:

```ts
export const studentDataResolver = resolve<Student, HookContext>({
  name: async (value, data, context) => {
    const originalData = await app.service('students').get(context.id)
    return value + ` (${originalData.gender})`
  }
})
```

> Note that you would actually want ot wrap the `.get` query in a try-catch, since it would throw an error in the case that an object with `context.id` doesn't exist on the database.



In the event that the resolver requires many references to the original data, you might be wondering if there is a way to retrieve ths original data once and save it somewhere, instead of having to perform a `.get` in every single resolver function that requires it. There are two ways to do this: the `converter` function, and Hooks. I actually think that the latter is the best way to address this, but let's talk about the `converter` function for the sake of completeness.



### The `converter` function

At this point, its important to at least know the existence of the `converter` function of a resolver. This function's job is to basically take the input data, and convert it to another form, to be used as the `instance` of the async functions of the resolver. The official documentation has a [good example](https://feathersjs.com/api/schema/resolvers.html#options) of this. 

However, the downside of using the `converter` function is that it breaks the typing quite a bit. Because the transformation of data is not defined anywhere, the typing of `instance` does not get updated to reflect what the `converter` functionr returns. For this reason, I think that the `converter` function should be used sparingly.



### Bonus Action (1)

Try throwing an error in one of the resolver functions. What is the shape of the error returned?
