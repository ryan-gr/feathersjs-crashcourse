# Chapter 2

### Validators

A validator is a function that takes in data, and performs validation on each field of that object. In Feathers, you would add the required validations to the TypeBox type definitions, as shown below

```ts
export const studentDataSchema = Type.Object(
  {
    name: Type.String({ minLength: 1, maxLength: 20 }),
    gender: Type.Union([Type.Literal('M'), Type.Literal('F')]),
    age: Type.Number({ minimum: 0, exclusiveMaximum: 100, multipleOf: 1 }),
    
    favouriteAuthor: Type.Optional(Type.String()),
    favouriteBook: Type.Union([Type.String(), Type.Null()]),
    favouriteQuote: Type.Optional(Type.Union([Type.String(), Type.Null()]))
  },
  { $id: 'StudentData', additionalProperties: false }
)
```

Given the above data schema, Feathers will look at given `data` during a create (POST), and ensure that each field follows the conditions set in the schema.



##### Required and Optional Fields

Note that by default, entries in schemas are considered `required`. To make a value optional, we either wrap the type in a `Type.Optional`, or declare it as a union of the intended type and `Type.Null`. However, note that they are slightly different. 

For a field declared in theh former method (such as with `favouriteAuthor`), you can leave out the field entirely when creating an object. For a field declared in the latter way (such as with the field `favouriteBook`), you have to pass in a valid or `null` value. While this is less convenient, it is required if we want to "clear" the data in the future (this will be discussed in "Clearing Values").

However, to simplify everything, we can define a new, custom type, `OptionalNullable`. This will allow us to not need to populate the field when passing in the data, but will also allow us to pass in `null` later on if we want to "clear" the field. 

```ts
export const OptionalNullable = <T extends TSchema>(input: T) => {
  return Type.Optional(Type.Union([input, Type.Null()]))
}
```

*editor's note: i feel like this should have been included in Feathers / TypeBox by default, but oh well*



##### Clearing Values

Imagine we have the the following schema:

```ts
export const studentDataSchema = Type.Object(
  {
    name: Type.String({ minLength: 1, maxLength: 20 }),
    bankDetails: Type.Object({
      name: Type.String(),
      bankNumber: Type.String()
    })
  },
  { $id: 'StudentData', additionalProperties: false }
)

export const studentPatchSchema = Type.Partial(studentDataSchema,
  { $id: 'StudentPatchData', additionalProperties: false }
)
```

> Pay some attention to how the patch schema is defined. It is defined as a Partial of the data schema (if needed, please read and understand what are `partial` types in Typescript).



And when we create the student object, we pass in the following data:

```json
{
  "name": "Ryan",
  "bankDetails": {
    "name": "DBS",
    "bankNumber": "1234-5678"
  }
}
```

If we want the user to be able to "delete" the `bankDetails` field (for example, to remove their personal data from our platform), we might try to patch with the following data:

```json
{
  "bankDetails": undefined
}
```

By default, Feathers ignores any field marked as `undefined`, hence the above patch would do nothing. To remove a field,  we need to pass `null` to it.

```json
{
  "bankDetails": undefined
}
```

However, if you were to use the above data and patch schemas, the above patch would still fail with a validation error (because we have defined `bankDetails` to only possibly be an object with two fields, `name` and `bankNumber`. In order to tell Feathers we want to be able to pass `null` to the field *in a patch*, we can use the `OptionalNullable` type mentioned above.

```ts
export const studentDataSchema = Type.Object(
  {
    name: Type.String({ minLength: 1, maxLength: 20 }),
    bankDetails: Type.Object({
      name: Type.String(),
      bankNumber: Type.String()
    })
  },
  { $id: 'StudentData', additionalProperties: false }
)

export const studentPatchSchema = Type.Object(
  {
    name: Type.String({ minLength: 1, maxLength: 20 }),
    bankDetails: OptionalNullable(Type.Object({
      name: Type.String(),
      bankNumber: Type.String()
    }))
  },
  { $id: 'StudentPatchData', additionalProperties: false }
)
```

Of course, if we wanted to also indicate that the field `bankDetails` might not exist, we would also need to add the `OptionalNullable` to the field in the data schema. In that case, we can just add it to the data schema, and define the patch schema to just be a partial of the data schema as at the beginning of this section!

```ts
export const studentDataSchema = Type.Object(
  {
    name: Type.String({ minLength: 1, maxLength: 20 }),
    bankDetails: OptionalNullable(Type.Object({
      name: Type.String(),
      bankNumber: Type.String()
    }))
  },
  { $id: 'StudentData', additionalProperties: false }
)

export const studentPatchSchema = Type.Partial(studentDataSchema,
  { $id: 'StudentPatchData', additionalProperties: false }
)
```

> **A small note on Feather's default schemas**
>
> Be default, Feathers sets up the the schemas in a certain way: the data schema is a `Pick` (or subset) of the main model schema, and the patch schema is a `Partial` of the data schema, while the query schema is a combination of a `Pick` of the main model schema and another user-defined object.
>
> One thing to note is that this is just the default configuration, but it might not always be the best for the service. For example, each schema could be defined independantly of each other, or in any other configuration. Notice that in chapter 1, we defined the main model and data schemas independantly, in order to more easily demonstrate how schemas work (*in my personal opinion*)



### Bonus Action (1)

Do some research on other types in TypeBox. Some that are useful to understand are: `Type.Literal`, `Type.Partial`, `Type.Pick`, `Type.Omit`, `Type.Intersect`, `Type.Union`, `Type.Array`,and `Type.Object`.



### Bonus Action (2)

With a type of `Type.String()`, try posting data to that field with the value `""`. Is any validation error thrown? Why? How do we make it such that empty strings are disallowed? (Hint, the length of the string).
