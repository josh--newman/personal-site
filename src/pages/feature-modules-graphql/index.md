---
title: Creating feature modules for your GraphQL API
date: "2018-07-29T02:11:38.146Z"
---

I've spent a fair bit of time building GraphQL APIs. More specifically, I've spent a fair bit of time building NodeJS + Express + Apollo Server GraphQL API stacks.

A new project starts out relatively easily. In the beginning, it's easy to get going quickly by defining your `typeDefs` and `resolvers` in a single file which you can then give to Apollo Server. Something like this:

```javascript
const typeDefs = gql`
  type Query {
    albums: [Album!]!
  }

  type Mutation {
    createAlbum(input: AlbumInput!): Album
  }

  type Album {
    title: String!
    released: String!
    artist: Artist!
  }

  type Artist {
    name: String!
  }
`

const resolvers = {
  Query: {
    album: () => fetch('https://album-api/albums')
  },
  Mutation: {
    createAlbum: (root, args) => fetch('https://album-api/albums/create',
      { method: 'POST', body: JSON.stringify(args.input) }
    )
  },
  Album: {
    title: album => album.name,
    released: album => album.yearReleased,
    artist: album => fetch(`https://artist-api.com/artist/${album.artistId}`)
  },
  Artist: {
    name: artist => artist.name
  }
}
```

#### The problem

This gets out of control very quickly. Having your whole schema in one place is fine while it's small, but as it grows you'll want to split it out into some logical grouping.

Your first instinct might be to split your typeDefs and resolvers into two different files. Again, this might be ok for a while, but it will soon get out of hand. What would be really nice is if we could group our type definitions and the resolvers for those types into their own **feature modules**, and then have some way to pull them all together so Apollo Server can parse them.

#### Defining feature modules

Let's go back to our previous `Album` and `Artist` example. Let's put all our Album related stuff (the `album` query, the `Album` type def, and the `Album` resolvers) into one folder, and all the `Artist` related stuff into another folder.

In order to make it easy to combine these **feature modules** later on, we need to define a consistent _interface_ for them:
```
Album/
|-- index.js (Exports the files in this module.)
|-- mutations.js (All top level mutations for this module.)
|-- queries.js (All top level queries for this module.)
|-- resolvers.js (All resolvers for types defined in schema.js.)
|-- schema.js (All types defined as part of this module.)
```

Let's look at these files individually.

##### index.js
```javascript
const schema = require('./schema');
const resolvers = require('./resolvers');
const { queries, queryDefs } = require('./queries');
const { mutations, mutationDefs } = require('./mutations');

module.exports = {
  name: 'Album',
  schema,
  resolvers,
  queries,
  queryDefs,
  mutations,
  mutationDefs
}
```

This file is just responsible for exposing the various files in a consistent interface. We can then import these modules later by just requiring the folder.

##### schema.js
```javascript
const schema = `
  type Album {
    title: String!
    released: String!
    artist: Artist!
  }
`;

module.exports = schema;
```

This is our type definition for the `Album` type, not including the top level `album` query.

##### resolvers.js
```javascript
const resolvers = {
  Album: {
    title: album => album.name,
    released: album => album.yearReleased,
    artist: album => fetch(`https://artist-api.com/artist/${album.artistId}`)
  }
}

module.exports = resolvers;
```

These are our resolvers for the `Album` type.

##### queries.js
```javascript
module.exports = {
  queryDefs: `
    albums: [Album!]!
  `,

  queries: {
    album: () => fetch('https://album-api/albums')
  }
}
```

This file contains both the `album` query definition, and the resolver for that query.

##### mutations.js
```javascript
module.exports = {
  mutationDefs: `
    createAlbum(input: AlbumInput!): Album
  `,

  mutations: {
    createAlbum: (root, args) => fetch('https://album-api/albums/create',
      { method: 'POST', body: JSON.stringify(args.input) }
    )
  }
}
```

This file contains both the `album` mutation definition, and the resolver for that mutation.

#### Combining all the modules
Now that we've defined a consistent interface for our feature modules and we're exposing them correctly, let's figure out a way to pull all the modules together in a format that Apollo Server can parse.

As seen in the first example, Apollo Server likes to receive all the schema type definitions and resolvers in a single object like so:
```javascript
{
  typeDefs: gql`
    type Query {
      # All the query definitions here
    }

    type Mutation {
      # All the mutation definitions here
    }

    # ... all of your custom type definitions here
  `,

  resolvers: {
    Query: {
      // All your query resolvers here
    },
    Mutation: {
      // All your mutation resolvers here
    },
    // ... all your custom type resolvers here
  }
}
```

It would be really convenient if we could just require each module once and give all the modules to a function that would know how to combine them into the object above. Let's see what that would look like.

##### loadModules.js
```javascript
const loadModules = (modules) => {
  const resolvers = modules.reduce((acc, m) => {
    return {
      ...acc,
      Query: {
        ...acc.Query || {},
        ...m.queries
      },
      Mutation: {
        ...acc.Mutation || {},
        ...m.mutations
      },
      ...m.resolvers
    }
  }, {});

  const schema = `
    type Query {
      ${modules.reduce((s, m) => s.concat(m.queryDefs), '')}
    }
    type Mutation {
      ${modules.reduce((s, m) => s.concat(m.mutationDefs), '')}
    }
  `;

  const typeDefs = [schema].concat(modules.map(m => m.schema));

  return {
    typeDefs,
    resolvers
  }
}

module.exports = loadModules;
```

Let's unpack this one thing at a time.

##### Resolvers
```javascript
const resolvers = modules.reduce((acc, m) => {
    return {
      ...acc,
      Query: {
        ...acc.Query || {},
        ...m.queries
      },
      Mutation: {
        ...acc.Mutation || {},
        ...m.mutations
      },
      ...m.resolvers
    }
  }, {});
```

Here we're doing a `reduce` on the modules array and building up the `resolvers` object. There's a lot of object spread operators here, nothing too fancy. We should expect the final result to look something like this:
```javascript
{
  Query: {
    album: () => fetch('https://album-api/albums')
  },
  Mutation: {
    createAlbum: (root, args) => fetch('https://album-api/albums/create',
      { method: 'POST', body: JSON.stringify(args.input) }
    )
  },
  Album: {
    title: album => album.name,
    released: album => album.yearReleased,
    artist: album => fetch(`https://artist-api.com/artist/${album.artistId}`)
  }
}
```

##### Query and Mutation definitions
```javascript
const schema = `
  type Query {
    ${modules.reduce((s, m) => s.concat(m.queryDefs), '')}
  }
  type Mutation {
    ${modules.reduce((s, m) => s.concat(m.mutationDefs), '')}
  }
`;
```

Here we're building our schema, which is just a big template literal. This part is just focused on concatenating the `Query` and `Mutation` parts, which `reduce` each module's `queryDefs` and `mutationDefs` into a single template literal respectively. The final result should look like this:
```javascript
`
  type Query {
    albums: [Album!]!
  }

  type Mutation {
    createAlbum(input: AlbumInput!): Album
  }
`
```

##### Custom type definitions
```javascript
const typeDefs = [schema].concat(modules.map(m => m.schema));
```

This line takes the `schema` result from above and concatenates the rest of the type definitions as defined in the `schema` module property. The final `typeDefs` result should look like this:
```javascript
`
  type Query {
    albums: [Album!]!
  }

  type Mutation {
    createAlbum(input: AlbumInput!): Album
  }

  type Album {
    title: String!
    released: String!
    artist: Artist!
  }
`
```

Finally, we return the object in the format that Apollo Server expects:
```javascript
return {
  typeDefs,
  resolvers
}
```

#### Putting it all together
Now that we have a way to combine all our modules together we can write a simple file that wraps up all the feature modules so we can pass it to Apollo:

##### `modules.js`
```javascript
const { makeExecutableSchema } = require('graphql-tools');
const loadModules = require('./loadModules');

// Our feature module folders
const Album = require('./Album');
const Listing = require('./Artist');

// Add more modules to here as your project evolves
const modules = [
  Album,
  Artist
];

const schema = loadModules(modules);

module.exports = makeExecutableSchema(schema);
```

#### Takeaways
In summary, organising your project into feature modules has a few key benefits:
- Provides a logical grouping of schema and resolver parts.
- Makes it easier to reason about your project structure.
- Adding new things to your schema becomes easier.

Here's a link to the [full example source code]() for your reference.

Let me know if you have any suggestions on how to make this better, or any other alternative ways to structure a GraphQL project like this.
