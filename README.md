# Prisma ORM Tutorial

In this tutorial we'll cover how to set up a GraphQL Server using the new Prisma ORM and resolver types generation in TypeScript **from scratch**. We'll follow a schema-driven development approach where we first define our schema
and let the nice new tooling from Prisma generate the GraphQL API and resolvers for us.

## 1. yarn init & install dependencies

```bash
mkdir orm-tutorial
cd orm-tutorial
yarn init -y
yarn add --dev prisma@alpha graphql-resolver-codegen@beta ts-node typescript
yarn add prisma-lib graphql graphql-yoga
```

## 2. initialize prisma service

```bash
mkdir prisma src
touch prisma/{prisma.yml,datamodel.graphql}
```

## 3. create prisma.yml

Write this into your `prisma.yml` you just created:

```yaml
datamodel: datamodel.graphql
generate:
  - generator: typescript
    output: ../src/generated/prisma.ts
```

## 4. create your datamodel

Now it's time to get creative. In your `datamodel.graphql` file you can create ANY
datamodel that you can think of. It can be a very simple one like
[this](https://github.com/prisma/orm-examples/blob/master/minimal-typescript/datamodel.graphql), but also a huuge one
like [our airbnb example](https://github.com/prisma/graphql-server-example/blob/orm/prisma/datamodel.graphql).

In this tutorial we want to use the following datamodel:

```graphql
type Cat {
  id: ID! @unique
  name: String!
  color: String!
  favBrother: Cat
}

type Master {
  id: ID! @unique
  catz: [Cat!]!
}
```

Please add it to `datamodel.graphql`

## 5. create your application schema

As we're following a schema-driven approach, we first need to define the GraphQL schema that our application will expose.
Let's create the schema file:

```
touch src/schema.graphql
```

And again for this tutorial, here is an application schema that we want to use:

```graphql
type Query {
  masters: [SpecialMaster!]!
}

type SpecialMaster {
  id: String!
  catBrothers: [Cat!]!
}

type Cat {
  id: ID!
  name: String!
  color: String!
  favBrother: Cat
}
```

## 5. let's generate

After having defined both the datamodel and the schema, let's start by generating the ORM and the resolvers.

For this we need the two tools `prisma` and `graphql-resolver-codegen`.
Please add them to your `package.json` like this:

```json
"scripts": {
  "start": "ts-node src/index.ts",
  "prisma": "cd prisma && prisma",
  "graphql-resolver-codegen": "graphql-resolver-codegen"
}
```

This now allows us to run `yarn prisma` and have a nice isolated version of the Prisma cli running locally without interfering with your global dependencies.

### 5.1 prisma deploy

In order to generate types, please first deploy your prisma service.

```bash
$ yarn prisma deploy
```

-> Please choose the target of your deployment

### 5.2 prisma generate

Now as we have a service deployed with our schema, let's generate the types for the ORM:

```bash
$ yarn prisma generate
```

### 5.3 graphql-resolver-codegen scaffold

Based on the `src/schema.graphql` that we created in step 5, we now generate our basic GraphQL resolvers.

```bash
$ yarn graphql-resolver-codegen interfaces -s src/schema.graphql
$ yarn graphql-resolver-codegen scaffold -s src/schema.graphql
```

## 6. add src/index.ts

Now we have our resolvers setup generated! Let's create the GraphQL Server entrypoint.

Create a file `src/index.ts` with the following content:

```ts
import { GraphQLServer } from 'graphql-yoga'
import { resolvers } from './resolvers'
import { Prisma } from './generated/prisma'

const server = new GraphQLServer({
  typeDefs: './src/schema.graphql',
  resolvers,
  context: {
    db: new Prisma({ debug: true }),
  },
} as any)

server.start(() => console.log('Server is running on localhost:4000'))
```

## 7. first run

Let's try out our new server!

```bash
yarn start
```

You now can try out your server on [http://localhost:4000](http://localhost:4000).

A query you could try out is the following:

```graphql
{
  masters {
    id
  }
}
```

But wait! You will receive an error `Resolver not implemented` 😱
What now?
Let's implement the resolvers!

## 8. implementing resolvers

### 8.1 Cat

Let's start by implementing our Cat resolver. We do this in `src/resolvers/Cat.ts`.
Our job now is to implement all relation fields - the scalars will be resolved automatically.
The first relation we implement is `favBrother`. The current implementation looks like this:

```ts
favBrother: root => root.favBrother
```

We turn this into

```ts
  favBrother: (root, args, ctx) =>
    ctx.db.cat({ id: root.id }).favBrother(),
```

Now, please don't just copy this over, but try to type it by hand.
You will realize, that there is **no autocompletion**! But isn't that the whole point of code generation?
Exactly! To get autocompletion we need to perform one more step: Add typings to the Context object. What the context is, we explain in a second.

The file `src/resolvers/Context.ts` looks like this right now:

```ts
export interface Context {
  db: any
  request: any
}
```

Please turn it into this:

```ts
import { Prisma } from '../generated/prisma'

export interface Context {
  db: Prisma
  request: any
}
```

Now as we added types for the Context, let's go back to `Cat.ts` and finish it, if you didn't already do it.
You will now see that you have nice autocompletion and linting for your input!

Ok, let's break down what we just wrote in `Cat.ts`. Resolvers written with the `graphql-js` library always have the same function signature:

1. `root` - this is the parent object that our field resolver belongs to
2. `args` - these are the args passed in into our field (if there are any)
3. `ctx` - this is a Context object, where we as application developers can choose a context for the resolvers that we can use to implement our GraphQL Server. If you check in `src/index.ts` - you will see that we added the ORM bindings on the property `db`. So from now on we can use the bindings with `ctx.db.fieldname()`

And let's break down what `ctx.db.cat({ id: root.id }).favBrother()` means. As we just said, `ctx.db` is the ORM binding that we just configured in `src/index.ts`. As you might remember from our `datamodel.graphql` - we have the type `Cat` defined. Prisma generates single queries and list queries for each type. In this case we want to resolve the `favBrother` of a single `Cat`, so we use the top-level single query `.cat`. In order to retrieve the correct cat, we can use `root.id`. This means that we can trust that the `id` has already been resolved. Otherwise TypeScript would let us know and tell use, that there is no `id` on the `root` object. Now to retrieve the `favBrother`, we just do another function call on `favBrother`. We don't have any args on this field - so we don't need to provide them.

Congratulations! You just wrote your first resolver. Just a few are left...

### 8.2 SpecialMaster

In `src/resolvers/SpecialMaster.ts` we now implement the `catBrothers` resolver.
The resolver becomes this:

```ts
catBrothers: (root, args, ctx) => ctx.db.master({ id: root.id }).catz()
```

This is basically the same structure as in the `favBrother` resolver we wrote for the `Cat` type. The only difference is, that you will get a list of `Cat`s instead of a single item. But as you see, you don't have to bother about that!

> Hint: Every time you don't know which api is available, just CMD+Click on the method and you'll get to the typings!

What we need to do now, though is the following: Remove `catBrothers: CatRoot[]` in line 7. We need to remove this, as we're resolving this by hand and our underlying Prisma schema doesn't contain the `catBrothers` field.

#### 8.3 Query

Last but not least, we implement the `masters` resolver in `src/resolvers/Query.ts`.

This is our `Query` right now:

```ts
export const Query: IQuery.Resolver<Types> = {
  masters: root => {
    throw new Error('Resolver not implemented')
  },
}
```

We turn it into this:

```ts
export const Query: IQuery.Resolver<Types> = {
  masters: (root, args, ctx) => ctx.db.masters(),
}
```

Again, we recommend not copying it over, but typings it by hand - this way you really feel the benefits of the autocompletion.

## 9 profit

Now we can start to query! Restart your server with executing

```
yarn start
```

again!

As we see, there is no data yet. To add data, please execute the following mutation at your endpoint:
(You get the endpoint by executing `yarn prisma info`)

```graphql
mutation {
  createMaster(
    data: {
      catz: {
        create: [
          {
            name: "Hans"
            color: "blue"
            favBrother: { create: { name: "Bro", color: "purple" } }
          }
        ]
      }
    }
  ) {
    id
  }
}
```

If you query your endpoint at `http://localhost:4000` again - we have data! Yey! 🤠

Congratulations! 🚀🎉🤘 You just implemented a GraphQL Server with the latest tools!!

## 10 for the lazy people

The result of this tutorial can also be found [here](https://github.com/prisma/orm-examples/tree/master/orm-tutorial)

## Next Steps

Now as you're already familiar with the architecture you could for example implement a mutation that can create Cats.

Another possibility is of course choosing your own datamodel now! You can repeat steps 1-6 with any schema that you can think of.

Have fun!
