# GraphQl

- [Intro to GraphQL](#intro-to-graphql)
  - [Querying a Server](#querying-a-server)
    - [Aliases](#aliases)
    - [Fragments](#fragments)
    - [Directives](#directives)
    - [Inline Fragments](#inline-fragments)
    - [Meta Fields](#meta-fields)
  - [Mutations](#mutations)
  - [Schemas](#schemas)
    - [Query and Mutation Types](#query-and-mutation-types)
    - [Scalar Types](#scalar-types)
  - [Resolvers](#resolvers)
- [GraphQL.js](#graphql.js)
  - [Express And Graph.QL](#express-and-graph.ql)
  - [Querying From Client](#querying-from-client)
  - [Mutations and Input on the Client](#mutations-and-input-on-the-client)
  - [Authentication and Middleware](#authentication-and-middleware)]
- [GraphQL and React](#graphql-and-react)

# Intro to GraphQL

## Querying a Server

- The GraphQL api only exposes a single endpoint to the client.
  - Clients need to send more information to the server to express its data needs - this information is called a query
  - queries can have zero or more arguments - the queries and arguments must be specified in the schema

```
# example of query with argument
query PersonInfo{
  human(id: "123") {
    name
    height
  }
}
```

### Aliases

- Aliases let you rename the result of a field

```
# If there weren't aliases, the two 'hero' fields would conflict
query EpisodeHero{
  empireHero: hero(episode: EMPIRE) {
    name
  }
  jediHero: hero(episode: JEDI) {
    name
  }
}
```

### Fragmnts

- Fragments allow you to reuse queries

```
query HeroComparison{
  comparison1: hero(episode: EMPIRE) {
    ...comparisonFields
  }
  comparison2: hero(episode: JEDI) {
    ...comparisonFields
  }
}

fragment comparisonFields on Character {
  name
  appearsIn
  friends {
    name
  }
}
```

- fragments can access variables declared in the query

```
query HeroComparison($first: Int = 3) {
  comparison1: hero(episode: EMPIRE) {
    ...comparisonFields
  }
}

fragment comparisonFields on Character {
  name
  friendsConnection(first: $first) {
    totalCount
    edges {
      node {
        name
      }
    }
  }
}
```

### Directives

- Directives can be used to dynamically change the structure/shape of queries using variables
  - directives can be attached to a field or fragment inclusion
  - examples: `@include(if: Boolean)` and `@skip(if: Boolean)`

```
query Hero($episode: Episode, $withFriends: Boolean!) {
  hero(episode: $episode) {
    name
    friends @include(if: $withFriends) {
      name
    }
  }
}

# variables
{
  "episode": "JEDI",
  "withFriends": false
}
```

### Inline Fragments

- inline fragments can be used to access data on the underlying concrete type
  - ex: the `hero` query returns the type `Character` which can be either `Droid` or `Human`.
  - In direct selection, you can only ask for fields that exist on the `Character` interface, but using inline fragments, you can access fields on either `Droid` or `Human`

```
query HeroForEpisode($ep: Episode!) {
  hero(episode: $ep) {
    name
    ... on Droid {
      primaryFunction
    }
    ... on Human {
      height
    }
  }
}

# variables
{
  "ep": "JEDI"
}
```

### Meta Fields

- meta fields: you can request the name of the object type of a field using `__typename`

```
# meta fields example

{
  search(text: "an") {
    __typename
    ... on Human {
      name
    }
    ... on Droid {
      name
    }
    ... on Starship {
      name
    }
  }
}
```

## Mutations

- Mutations allow the client to modify the server-side data
- If the mutation returns an object type, you can ask for nested fields
  - useful for fetching the new state after an update

```
mutation CreateReviewForEpisode {
  createReview(episode: JEDI, review: "great") {
    review
  }
}
```

## Schemas

- Defining a schema: graphQL is strongly typed
  - `!` indicates the field is non-nullable
  - `[Post]` - is an array of Posts

```
# example schema
type Post {
  title: String!
  author: Person!
}

type Person {
  name: String!
  age: Int!
  posts: [Post!]!
}
```

- every field on a graphQl object type can have zero or more arguments
  - arguments are named and should be passed by name

```
type Person {
  name: String
  picture(size: Int): Url
}
```

### Query and Mutation types

- the query and mutation types are the **entry point** of every GraphQL query

```
# querying the server
query {
  hero {
    name
  }
  droid(id: "2000") {
    name
  }
}

# the GraphQL schema must have a Query type as follows
type Query {
  hero(episode: Episode): Character
  droid(id: ID!): Droid
}
```

### Scalar Types

- default scalar types include:
  - `Int`
  - `Float`
  - `String`
  - `Boolean`: `true` or `false`
  - `ID`: represents a unique identifier, it is serialized the same as a String
- you can specify custom service types
  - ex: `scalar Date`
- You can also specify `Enums` - special kind of scalar restricted to a set of allowed values

```
enum Episode {
  NEWHOPE
  EMPIRE
  JEDI
}
```

### Interfaces

- an **interface** is an abstract type that includes a certain set of fields that a type must include to implement the interface

```
interface Character {
  id: ID!
  name: String!
  friends: [Character]
  appearsIn: [Episode]!
}
```

- then your types can implement an interface
  - to implement the interface you must include the fields of the interface, but you can also bring in extra fields that are specific to that particular type of character

```
type Human implements Character {
  id: ID!
  name: String!
  friends: [Character]
  appearsIn: [Episode]!
  starships: [Starship]
  totalCredits: Int
}
```

### Input Types

- you can pass complex objects as arguments into a field
  - to do this, preface the object with `input` instead of `type`

```
input ReviewInput {
  stars: Int!
  commentary: String
}

# this allows us to pass the ReviewInput to a mutation defined in the schema
type Mutation {
  createReview(episode: Episode, review: ReviewInput): Review
}
```

## Resolvers

- resolver functions are required for each field on each type
- the resolver function produces the next value by accessing a data base
- resolvers have 4 arguments
  - `obj` - the previous object; for a field on the root Query type, this argument is often not used
  - `args` - the arguments provided to the field in the GraphQL query
  - `context` - holds contextual information (ex: currently logged in user, access to a database)
  - `info` - holds field-specific information relevant tot eh current query as well as the schema details
- resolvers in JavaScript
  - the `context` is used to provide access to the database
  - the loading from the database is asynchronous and returns a Promise

```
human(obj, args, context, info) {
  return context.db.loadHumanByID(args.id).then(
    userData => new Human(userData)
  )
}
```

# GraphQL.js

```
var { graphql, buildSchema } = require('graphql');

// Construct a schema, using GraphQL schema language
var schema = buildSchema(`
  type Query {
    hello: String
  }
`);

// The root provides a resolver function for each API endpoint
var root = {
  hello: () => {
    return 'Hello world!';
  },
};

// Run the GraphQL query '{ hello }' and print out the response
graphql(schema, '{ hello }', root).then((response) => {
  console.log(response);
});
```

## Express and GraphQL

- uses npm packages `express` `express-graphql` `graphql`
- using express with graphql allows you to run graphql queries from an api server

## Querying From Client

```
fetch('/graphql', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  },
  body: JSON.stringify({query: "{ hello }"})
})
  .then(r => r.json())
  .then(data => console.log('data returned:', data));
```

- you can use variables as follows:

```
var dice = 3;
var sides = 6;
var query = `query RollDice($dice: Int!, $sides: Int) {
  rollDice(numDice: $dice, numSides: $sides)
}`;

fetch('/graphql', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  },
  body: JSON.stringify({
    query,
    variables: { dice, sides },
  })
})
  .then(r => r.json())
  .then(data => console.log('data returned:', data));
```

## Mutations and Inputs on the Client

- similarly to queries, mutations are handled by the root resolver

```
type Mutation {
  setMessage(message: String): String
}

type Query {
  getMessage: String
}
```

```
var fakeDatabase = {};
var root = {
  setMessage: ({message}) => {
    fakeDatabase.message = message;
    return message;
  },
  getMessage: () => {
    return fakeDatabase.message;
  }
};
```

- when calling the mutation from the client, the call must be prefaced by `mutation` as opposed to `query`

```
var author = 'andy';
var content = 'hope is a good thing';
var query = `mutation CreateMessage($input: MessageInput) {
  createMessage(input: $input) {
    id
  }
}`;

fetch('/graphql', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  },
  body: JSON.stringify({
    query,
    variables: {
      input: {
        author,
        content,
      }
    }
  })
})
  .then(r => r.json())
  .then(data => console.log('data returned:', data));
```

## Authentication and Middleware

- you can use express middleware with `express-graphql`

```
var express = require('express');
var graphqlHTTP = require('express-graphql');
var { buildSchema } = require('graphql');

var schema = buildSchema(`
  type Query {
    ip: String
  }
`);

const loggingMiddleware = (req, res, next) => {
  console.log('ip:', req.ip);
  next();
}

var root = {
  ip: function (args, request) {
    return request.ip;
  }
};

var app = express();
app.use(loggingMiddleware);
app.use('/graphql', graphqlHTTP({
  schema: schema,
  rootValue: root,
  graphiql: true,
}));
app.listen(4000);
console.log('Running a GraphQL API server at localhost:4000/graphql');
```

- for authentication, check `express-jwt` middleware

# GraphQL and React

- GraphQL uses query strings that are interpreted by a server, which returns data in a format specified by those queries
  - the queries are in a JSON like format and the response is JSON
