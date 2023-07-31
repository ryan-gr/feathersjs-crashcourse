# Chapter 1

### Initial Setup

1. Create a MongoDB Atlas database - we will be using MongoDb as the main database of this project.

   a. Take note of the connection string of the new database

2. Create a new folder named `my-server` that will hold the new project

3. Run the following command to generate a new Feathers App 

   `npx @feathersjs/cli generate app`

4. Use the default options, except for choosing MongoDb as a database, as well as entering the database connection string.
5. The Feathers CLI will generate a new app for us to use!

While you don't actualy need to run the server for this exercise, the command for that is:

`npm run dev`

or

`npm run compile`

`npm run start`

### Generating a service

For a start, let's explore how we can use Feathers to create a simple API, that will create and manage entries in a table in our database. For the purposes of this exercise, we will be creating an endpoint that will handle the recording of student's name and details. In Feathers, we create a `service` to handle this. Use the command below to create the `students` service.

`npx feathers generate service`

What is the name of your service? student
Which path should the service be registered on? student**s**
Does this server require authentication? No
What database is the service using? MongodDB
Which schema definition format do you want to use? TypeBox

> Note what files were generated. Most should be in the `src/services/students` folder, but the files `src/client.ts` and `src/services/index.ts` were also updated to include our new service. 



### Schemas

When creating an API endpoint, we typically have to ask ourselves some questions:

1. What type of data does it accept (during a `create`)?
2. What type of data does it return (during a `get` or `find`)?
3. What type of query parameters does it accept (during a `get` or `find`)?

In Feathers, we define the answer to these things in a `.schema.ts` file. In the schema file, Feathers has already generated a default set of schemas to use. Here is a table describing what each schema is used for.

| Schema                            | Function                                                     |
| --------------------------------- | ------------------------------------------------------------ |
| Main Model Schema (studentSchema) | Determines the data type that operations like `get` or `find` will return. I.e. "if I ask the service to describe a Student object, how will it look like?" |
| Data Schema (studentDataSchema)   | Determines the data type that is accepted by `create` (i.e. a POST). Importantly, this describes **only** the data that the provided to Feathers, but might not be what is actually saved in the database. |
| Patch Schema (studentPatchSchema) | Determines the data type accepted by a `patch`.              |
| Query Schema (studentQuerySchema) | Determines the query parameters accepted by the service. Most often this is used during a `find`, but can be used in all other methods as well. |



For the sake of this exercise, lets use the below as the schema file

```ts
// // For more information about this file see https://dove.feathersjs.com/guides/cli/service.schemas.html
import { resolve } from '@feathersjs/schema'
import { Type, getValidator, querySyntax } from '@feathersjs/typebox'
import { ObjectIdSchema } from '@feathersjs/typebox'
import type { Static } from '@feathersjs/typebox'

import type { HookContext } from '../../declarations'
import { dataValidator, queryValidator } from '../../validators'

// Main data model schema
export const studentSchema = Type.Object(
  {
    _id: ObjectIdSchema(),
    name: Type.String(),
    gender: Type.Union([Type.Literal('M'), Type.Literal('F')]),
    createdAt: Type.Number()
  },
  { $id: 'Student', additionalProperties: false }
)
export type Student = Static<typeof studentSchema>
export const studentValidator = getValidator(studentSchema, dataValidator)
export const studentResolver = resolve<Student, HookContext>({})

export const studentExternalResolver = resolve<Student, HookContext>({})

// Schema for creating new entries
export const studentDataSchema = Type.Object(
  {
    name: Type.String(),
    gender: Type.Union([Type.Literal('M'), Type.Literal('F')])
  },
  { $id: 'StudentData', additionalProperties: false }
)
export type StudentData = Static<typeof studentDataSchema>
export const studentDataValidator = getValidator(studentDataSchema, dataValidator)
export const studentDataResolver = resolve<Student, HookContext>({})

// Schema for updating existing entries
export const studentPatchSchema = Type.Partial(studentSchema, {
  $id: 'StudentPatch'
})
export type StudentPatch = Static<typeof studentPatchSchema>
export const studentPatchValidator = getValidator(studentPatchSchema, dataValidator)
export const studentPatchResolver = resolve<Student, HookContext>({})

// Schema for allowed query properties
export const studentQueryProperties = Type.Pick(studentSchema, ['_id', 'name', 'gender', 'createdAt'])
export const studentQuerySchema = Type.Intersect(
  [
    querySyntax(studentQueryProperties),
    // Add additional query properties here
    Type.Object({}, { additionalProperties: false })
  ],
  { additionalProperties: false }
)
export type StudentQuery = Static<typeof studentQuerySchema>
export const studentQueryValidator = getValidator(studentQuerySchema, queryValidator)
export const studentQueryResolver = resolve<StudentQuery, HookContext>({})

```

Try to look at the above and answer these questions:

1. What type of data does the service accept?
2. What type of data is expected to be returned from the service?
3. What type of query params are we expecting to be able to use?



In order to try the effects of the schemas, open the `student.tests` file and override it with the following:

```ts
// For more information about this file see https://dove.feathersjs.com/guides/cli/service.test.html
import assert from 'assert'
import { app } from '../../../src/app'

describe('students service', () => {
  it('registered the service', async () => {
    const service = app.service('students')

    const newStudent = await service.create({
      name: 'Ted',
      gender: 'M'
    })

    assert.ok(service, 'Registered the service')
  })
})

```

> Try changing the values of `gender`, as well looking at how the `newStudent` object is typed



### Bonus Action (1)

We want to add at what time the `student` object was created. Add a `createdAt` field to the main modal schema, as well as a simple data [resolver](https://feathersjs.com/api/schema/resolvers.html) to populate the field with `Date.now()` when the object is created.



### Bonus Action (2)

After creating a few `student` entries, try returning them as a list of students (either through the test file above, or postman). Try to return a list that is sorted by `name`, or by `createdAt`, using filters described [here](https://feathersjs.com/api/databases/querying.html). Explore how to limit the number of entries returned, as well as the number of entries skipped.



### Query Schemas

The Query Schema actually looks quite different from the other schemas, so let's take some time to look at it specifically.

```ts
export const studentQueryProperties = Type.Pick(studentSchema, ['_id', 'name', 'gender', 'createdAt', 'age'])
export const studentQuerySchema = Type.Intersect(
  [
    querySyntax(studentQueryProperties),
    // Add additional query properties here
    Type.Object({}, { additionalProperties: false })
  ],
  { additionalProperties: false }
)
```

On the first line, we see something familiar - we `Pick` some fields defined in the `studentSchema`, the main model of the service.

On the fourth line, we are introduced to `querySyntax`. This function basically "expands" the given fields to also accept common operators for [querying](https://feathersjs.com/api/databases/querying.html), such as `$lt`, `$lte`, `$gt`, `$gte`, `$ne`, `$in` and `$nin`.

For example, even though we defined `age` as a `Type.Number`, when using `age` in a search query, we are not limited to just using: `{ query: { age: 5} }`, but we could use `{ query: { age: { $gte: 5 } } }`, or even `{ query: { age: { $in: [1,5, 10] } } }`. 

In the event that you would want to defined custom, additional fields to a queried field, you could do the following:

```ts
export const studentQueryProperties = Type.Pick(studentSchema, ['_id', 'name', 'gender', 'createdAt', 'age'])
export const studentQuerySchema = Type.Intersect(
  [
    querySyntax(studentQueryProperties, {
      name: { $regex: Type.String() }
    }),
    // Add additional query properties here
    Type.Object({}, { additionalProperties: false })
  ],
  { additionalProperties: false }
)
```

The above would allow us to query name like so: `{ query: { name: { $regex: 'ryan', } } }`, as well as the other ways to query a string that were already available to us (which includes the common operations mentioned above). Note that some operators can only be used for certain types of data though.

If you still require more flexibility, you could define totally new query paramters as well, such as:

```ts
export const studentQueryProperties = Type.Pick(studentSchema, ['_id', 'name', 'gender', 'createdAt', 'age'])
export const studentQuerySchema = Type.Intersect(
  [
    querySyntax(studentQueryProperties, {
      name: { $regex: Type.String() }
    }),
    // Add additional query properties here
    Type.Object({
      isTeenager: Type.Optional(Type.Boolean())
    }, { additionalProperties: false })
  ],
  { additionalProperties: false }
)
```

Note that most of the time when doing this, we don't actually want the query param to be converted to the actual query to be run on the database. For example, there is no `isTeener` field on the database that it can check against. What we could do is that override the or add  an `age` query, without actually passing the query from the client. This will be disussed in the next section regarding **Resolvers** (this could also be done with **Hooks**, again, a topic for another day).

> Not handling additional query properties properly could cause weird issues down the line. For example, if we don't remove the `isTeenager` query param somewhere down the line (either in a resolver or hook), the database would probably be giving you empty results, since none of your data actually has a `isTeenager` field.



##### Nested Field Querying (MongoDB Specific)

For nested data (especially important for NoSQL databases), we typically access them using dot notation. For example, `bankDetails.accountNumber`. However, if you try to do that with the default schemas, Feathers will expect you to provide an entire object of `bankDetails` in the query for comparison. 

In order to search for nested fields, we have to declare `bankDetails.accountNumber` as an additional query property, as shown below:

```ts
export const studentQueryProperties = Type.Pick(studentSchema, ['_id', 'name', 'gender', 'createdAt', 'age'])
export const studentQuerySchema = Type.Intersect(
  [
    querySyntax(studentQueryProperties),
    // Add additional query properties here
    Type.Object({
      'bankDetails.accountNumber': Type.Optional(Type.String())
    }, { additionalProperties: false })
  ],
  { additionalProperties: false }
)
```

In this case, no further configuration or consideration is needed for the query to work, since MongoDB actually accepts this dot notation as valid querying notation.



##### Querying for null, empty, or non-existant values (MongoDB specific)

Mongo is actually quite strict on querying for values that don't exist, are set to null, or are empty strings or empty arrays. The possible permutations of what is accepted and or not is definitely beyond the scope of this guide, and there are many of [resources](https://www.mongodb.com/docs/manual/tutorial/query-for-null-fields/) out there to correctly query for a certain set of conditions (including ChatGPT or Bard). But as a simple demonstration of the required considerations, consider the following POST data: 

```ts
const postData1 = {  name: "" }
const postData2 = {  name: null }
const postData3 = {  name: undefined }
```

The first two data would actually create a `name` field on the resulting database entry, one with an empty string `""`, and one with `null`. However, the third data would create an empty object, without the `name` field. 

In this scenerio, querying with `{ name: null }`  would return the 2nd and 3rd records. 

If we were to query using `{ name: { $exists: true } }`, it would return the 1st and 2nd records. 

For the query `{ name: "" }`, it would only return the 1st record.

If these were records amidst other properly-populated records, and we wanted to get all non-empty, non-null, and non-undefined records, we would have to use the query: `{ name: { $exists: true, $ne: '' } }`

This section is less of an instruction and more of a warning: **Great care must be taken in how empty strings, arrays, null values, and non-existant fields are queries.** 

> A very common query i find useful is for when I'm trying to find an optional boolean value that is "false". In the event that there exists records withou the field, a query `{ field: false }` will not return those records without any `field` field. I usually use the query `{ field: { $ne: true } }` to represent "false" in this context.
