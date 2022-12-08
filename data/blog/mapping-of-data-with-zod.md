---
title: 'Mapping of data with Zod'
date: '2022-12-08'
tags: ['api', 'zod', 'typescript', 'javascript', 'trpc', 'twitter', 'google']
draft: false
summary: 'Mapping data in Javascript can sometimes be hard so why not use a library like Zod to do the hard work?'
authors: ['default']
---

# The problem

Mapping data in Javascript/Typescript is not always easy. You need to create different kinds of checks(if/switch statements) to validate if your data has the shape x or y. You also need to find the difference between 2 shapes and use that difference to create your validation.

Another thing that will be hard is using the correct types(in Typescript), there are some tricks to do that but those will add extra complexity to your project.

In the end, you are creating a lot of extra code/complexity to make your mapper work. So what if you can use a library like [Zod](https://zod.dev/) to help with the mapping part?

```typescript
const user = {
  id: '123',
  firstName: 'Stijn',
  lastName: 'Van Hulle',
  email: 'stijn@stijnvanhulle.be',
} as const

const twitterUser = {
  id: '123',
  first: 'Stijn',
  last: 'Van Hulle',
  email: 'stijn@stijnvanhulle.be',
} as const

const googleUser = {
  uuid: '123',
  info: {
    firstName: 'Stijn',
    lastName: 'Van Hulle',
  },
  email: 'stijn@stijnvanhulle.be',
} as const

// no type support
const mapData = (data: any) => {
  if (data.info) {
    // google user
    return {
      id: user.uuid,
      value: user.info.firstName + ' ' + user.info.lastName,
    }
  } else if (data.first && data.last) {
    return {
      id: user.id,
      value: user.first + ' ' + user.last,
    }
  } else if (data.firstName && data.lastName) {
    return {
      id: user.id,
      value: user.firstName + ' ' + user.lastName,
    }
  }
}
mapData(googleUser) /* => {
        id: '123',
        value: 'Stijn Van Hulle'
    }
*/
```

# What is Zod?

Zod is described as: "TypeScript-first schema validation with static type inference "

You can use Zod to validate a form, create type-safe schema's and create type-safe API's. One example of a library that is using Zod is [TRPC](https://trpc.io/). With TRPC, you specify your schema once on the back-end. And then in the front-end, you can use the same schema to have a type-safe experience for doing API calls.

While there are many TypeScript [schema validation libraries](https://zod.dev/?id=comparison) out there, Zod has some extra features that can assist you in creating a type-safe environment.
Zod has out-of-the-box Typescript support which means you can create a schema and let Typescript infer the type based on the schema. All of this will be useful for creating a mapper based on Zod.

## Zod primitives

The following code is a simple example of how you can use Zod to check if a value is a string.

```typescript
import { z } from 'zod'

// creating a schema for strings
const mySchema = z.string()

// parsing
mySchema.parse('tuna') // => "tuna"
mySchema.parse(12) // => throws ZodError

// "safe" parsing (doesn't throw error if validation fails)
mySchema.safeParse('tuna') // => { success: true; data: "tuna" }
mySchema.safeParse(12) // => { success: false; error: ZodError }
```

## Zod object

Another thing you can do with Zod is creating objects and validate if the returned shape is equal to the one specified.

```typescript
import { z } from 'zod'

const User = z.object({
  username: z.string(),
})

User.parse({ username: 'Ludwig' })

// extract the inferred type
type User = z.infer<typeof User>
// { username: string }

User.parse({ firstname: 'firstname' }) // => throws ZodError
```

## Zod transform

Zod can also map an object/primitives from a specific shape to another shape.
The following example will return the origin of an email address.

```typescript
const emailToDomain = z
  .string()
  .email()
  .transform((val) => val.split('@')[1])

emailToDomain.parse('colinhacks@example.com') // => example.com
```

# Mapping data

## Simple Zod mapper

Now that we have a basic understanding of Zod, we can start with creating a mapper schema. We will use the basic primitives together with the transform functionality to map our data.

1. Let's say we have the following input and we want to map this to an object that only contains _id_ and _value_(full name of an user).

```typescript
const user = {
  id: '123',
  firstName: 'Stijn',
  lastName: 'Van Hulle',
  email: 'stijn@stijnvanhulle.be',
}
```

2. We can use the previously described transforms to map the user(containing _id_ and _value_).

```typescript
import z from 'zod'

export const userSchema = z
  .object({
    id: z.string(),
    firstName: z.string(),
    lastName: z.string(),
    email: z.string(),
  })
  .transform((user) => {
    return {
      id: user.id,
      value: user.firstName + ' ' + user.lastName,
    }
  })
```

3. The last thing we need to do is call _.parse_ with _user_ as the first parameter and that will return the mapped data.

```typescript
userSchema.parse(user) /* => {
        id: '123',
        value: 'Stijn Van Hulle'
    }
*/
```

## Multiple inputs mapper

But what if we have multiple inputs with different shapes/schemas?

```typescript
import z from 'zod'

const user = {
  id: '123',
  firstName: 'Stijn',
  lastName: 'Van Hulle',
  email: 'stijn@stijnvanhulle.be',
}

const twitterUser = {
  id: '123',
  first: 'Stijn',
  last: 'Van Hulle',
  email: 'stijn@stijnvanhulle.be',
}

export const userSchema = z
  .object({
    id: z.string(),
    firstName: z.string(),
    lastName: z.string(),
    email: z.string(),
  })
  .transform((user) => {
    return {
      id: user.id,
      value: user.firstName + ' ' + user.lastName,
    }
  })

export const twitterUserSchema = z
  .object({
    id: z.string(),
    first: z.string(),
    last: z.string(),
    email: z.string(),
  })
  .transform((user) => {
    return {
      id: user.id,
      value: user.first + ' ' + user.last,
    }
  })

userSchema.parse(user) /* => {
        id: '123',
        value: 'Stijn Van Hulle'
    }
*/

twitterUserSchema.parse(twitterUser) /* => {
        id: '123',
        value: 'Stijn Van Hulle'
    }
*/
```

## Multiple unknown inputs mapper

But what if we don't know what the user's shape will be?
Let's say we have 2 API calls:

- One to our database
- One to an external API(for example Twitter).

We can, of course, create some checks to see where the data is coming from but that will not make it scalable and we will have the same issue as before: we need to create if statements to check the input before we can convert it.

```typescript
import z from 'zod'

const user = {
  id: '123',
  firstName: 'Stijn',
  lastName: 'Van Hulle',
  email: 'stijn@stijnvanhulle.be',
}

const twitterUser = {
  id: '123',
  first: 'Stijn',
  last: 'Van Hulle',
  email: 'stijn@stijnvanhulle.be',
}

export const userSchema = z
  .object({
    id: z.string(),
    firstName: z.string(),
    lastName: z.string(),
    email: z.string(),
  })
  .transform((user) => {
    return {
      id: user.id,
      value: user.firstName + ' ' + user.lastName,
    }
  })

export const twitterUserSchema = z
  .object({
    id: z.string(),
    first: z.string(),
    last: z.string(),
    email: z.string(),
  })
  .transform((user) => {
    return {
      id: user.id,
      value: user.first + ' ' + user.last,
    }
  })

const mapData = (data: any) => {
  if (data.first) {
    // twitter
    twitterUserSchema.parse(data)
  } else {
    // database user
    return userSchema.parse(data)
  }
}

mapData(twitteruser) /* => {
        id: '123',
        value: 'Stijn Van Hulle'
    }
*/

mapData(user) /* => {
        id: '123',
        value: 'Stijn Van Hulle'
    }
*/
```

## Zod unions

This will do the trick but what if we have another source, for example, user data coming from Google?
Do we again want to create another if statement?

That feels hard to maintain so why not use the power of Zod and let Zod figure out what the shape and type will be?

In Zod you can use unions(also used with an .or function) and Zod will figure out the shape.

```typescript
const schema = z.string().or(z.number()) // string | number
// equivalent to
z.union([z.string(), z.number()])

stringOrNumber.parse('foo') // passes
stringOrNumber.parse(14) // passes
```

## End result

Combine that with transforms and you can use the power of Zod to do the hard work in finding the correct schema/shape.

So with our previous example, we can simplify that to the following:

```typescript
const user = {
  id: '123',
  firstName: 'Stijn',
  lastName: 'Van Hulle',
  email: 'stijn@stijnvanhulle.be',
} as const

const twitterUser = {
  id: '123',
  first: 'Stijn',
  last: 'Van Hulle',
  email: 'stijn@stijnvanhulle.be',
} as const

const googleUser = {
  uuid: '123',
  info: {
    firstName: 'Stijn',
    lastName: 'Van Hulle',
  },
  email: 'stijn@stijnvanhulle.be',
} as const
```

```typescript
import z from 'zod'

export const userSchema = z
  .object({
    id: z.string(),
    firstName: z.string(),
    lastName: z.string(),
    email: z.string(),
  })
  .transform((user) => {
    return {
      id: user.id,
      value: user.firstName + ' ' + user.lastName,
    }
  })

export const twitterUserSchema = z
  .object({
    id: z.string(),
    first: z.string(),
    last: z.string(),
    email: z.string(),
  })
  .transform((user) => {
    return {
      id: user.id,
      value: user.first + ' ' + user.last,
    }
  })

export const googleUserSchema = z
  .object({
    uuid: z.string(),
    info: z.object({
      firstName: z.string(),
      lastName: z.string(),
    }),
    email: z.string(),
  })
  .transform((user) => {
    return {
      id: user.uuid,
      value: user.info.firstName + ' ' + user.info.lastName,
    }
  })
```

```typescript
const mapData = (data: any) => twitterUserSchema.or(userSchema).or(googleUserSchema).parse(data)

export const users = [mapData(twitterUser), mapData(user), mapData(googleUser)]
```

[![Edit zod-mapper-object](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/jovial-worker-dl978e?fontsize=14&hidenavigation=1&theme=dark)

## Summary

The magic here is that you can add as many unions as you need(so long there is no overlap). Zod will throw an error if it cannot find a schema that is matching the input. This will make it scalable and in the end, you have still control over the types.

This approach can help you with creating mappers in Javascript but in the end, you should keep things as easy as possible. Using x amount of different schema's/types will make your application harder to maintain so think twice before using something like this.

Zod is really powerful so I would courage you in exploring what this library can do. I would suggest following this tutorial to have a better understanding of what Zod can do for your project: [https://www.totaltypescript.com/tutorials/zod](https://www.totaltypescript.com/tutorials/zod).
