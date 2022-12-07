---
title: 'Use Zod to transform your input'
date: '2022-12-08'
tags: ['api', 'zod', 'typescript']
draft: false
summary: 'Transform your input to a specific output without knowing what the type/schema of your input will be. Zod is working together with Typescript which mean when you call .parse() you will get back the correct output.'
authors: ['default']
---

# Why use Zod?

While there are many TypeScript schema validation libraries out there, Zod is one of the best. When choosing a library for your project, you should consider its implementation details in addition to its feature set.

Zod has zero dependencies which means that you can install and use Zod without any other libraries, and it will help you keep your bundle size smaller.

With [Zod](https://zod.dev/) you can als infer the Typescript type which makes it really powerful to use together with all kind of libraries.

# Introduction

You can use Zod to validate a form and it's also possible to use Zod as an alternative for GraphQL where you will have full control over the types used between front-end and back-end. One of those libraries is [https://trpc.io/](https://trpc.io/).

## Zod primitives

The following is a simple example that will do validation to check if the value is a string or not.

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

Another thing you can do with Zod is creating a object and check the value you pass through has the same shape as what is expected.

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

Zod also has the possibilties to map an object/primitives from shape x to shape y.
The following example will return the host of a specific email.

```typescript
const emailToDomain = z
  .string()
  .email()
  .transform((val) => val.split('@')[1])

emailToDomain.parse('colinhacks@example.com') // => example.com
```

# Map an input to a specific output

Let's say we have the following input and we want to map this to an object that only contains _id_ and _value_(a combination of firstName and lastName).

```typescript
import z from 'zod'

const user = {
  id: '123',
  firstName: 'Stijn',
  lastName: 'Van Hulle',
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

userSchema.parse(user) /* => {
        id: '123',
        value: 'Stijn Van Hulle'
    }
*/
```

What if we have multiple inputs?

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
      id: valuserue.id,
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

But what if we don't know what the user shape will be. Let's say we have 2 API calls: one with our own database an users and another one that will contain users coming from Twitter(external api). We can of course create some checks to see where the data is coming from.

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

const checkSource = (data: any) => {
  if (data.first) {
    //twitter use
    twitterUserSchema.parse(data)
  } else {
    // our own database user
    return userSchema.parse(data)
  }
}

checkSource(twitteruser) /* => {
        id: '123',
        value: 'Stijn Van Hulle'
    }
*/

checkSource(user) /* => {
        id: '123',
        value: 'Stijn Van Hulle'
    }
*/
```

This will do the trick but what if we have another source, for example user data coming from Google. Do we again want to create another check in our if? That feels hard to maintain so why not using the power of Zod and let Zod figure out what the shape and type will be?

In Zod you can use unions and Zod will figure out what matches and use that schema to do the parsing.

```typescript
const schema = z.string().or(z.number()) // string | number
// equivalent to
z.union([z.string(), z.number()])

stringOrNumber.parse('foo') // passes
stringOrNumber.parse(14) // passes
```

Combine that with transforms and you can use do mapping based on the input. So with our previous example we can simplify that to the following:

```typescript
import z from 'zod'

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

const checkSource = (data: any) => twitterUserSchema.or(userSchema).or(googleUserSchema).parse(data)

export const users = [checkSource(twitterUser), checkSource(user), checkSource(googleUser)]
```

The magic here is that you can add as many unions as you need(so long there is no overlap). Zod will also throw an error if it cannot find a schema that is matching the input. Feel free to try changing the data and see the validation message you will get back from Zod.

[![Edit zod-mapper-object](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/jovial-worker-dl978e?fontsize=14&hidenavigation=1&theme=dark)
