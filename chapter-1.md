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



### The Schema File

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
| Query Schema (studentQuerySchema) | Determinesi the query parameters accepted by the service. Most often this is used during a `find`, but can be used in all other methods as well. |



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
