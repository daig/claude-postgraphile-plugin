# Configuration, Presets, and Plugins

PostGraphile V5 (Release Candidate, October 2025) uses a preset-based configuration system. Presets bundle plugins and settings, and can be extended and combined.

> **Note:** V4 is now feature-frozen with security fixes only. New projects should use V5.

## Installation

Install PostGraphile V5 RC using the `@rc` tag:

```bash
# npm
npm install --save postgraphile@rc

# yarn
yarn add postgraphile@rc

# pnpm
pnpm add postgraphile@rc
```

Always use local installation (not global with `-g`). Run via `npx`, `yarn dlx`, or `pnpm dlx`.

## Configuration File

```js
// graphile.config.mjs
import { PostGraphileAmberPreset } from "postgraphile/presets/amber";
import { makePgService } from "postgraphile/adaptors/pg";

/** @type {import("postgraphile").GraphileConfig.Preset} */
const preset = {
  extends: [PostGraphileAmberPreset],

  pgServices: [
    makePgService({
      connectionString: process.env.DATABASE_URL,
      schemas: ["public"],
      pubsub: true,
    }),
  ],

  schema: {
    // Schema generation options
  },

  grafast: {
    // Execution options
  },

  grafserv: {
    // Server options
  },
};

export default preset;
```

## Built-in Presets

### PostGraphileAmberPreset (Recommended)

The standard V5 preset with sensible defaults:

```js
import { PostGraphileAmberPreset } from "postgraphile/presets/amber";

export default {
  extends: [PostGraphileAmberPreset],
};
```

Features:
- Node interface (Relay global IDs)
- Cursor-based pagination
- Condition and filter arguments
- Ordering
- CRUD mutations

### V4 Compatibility Preset

For migrating from V4:

```js
import { PostGraphileAmberPreset } from "postgraphile/presets/amber";
import { PostGraphileV4Preset } from "postgraphile/presets/v4";

export default {
  extends: [PostGraphileAmberPreset, PostGraphileV4Preset],
};
```

### Relay Preset

For Relay-focused schemas:

```js
import { PostGraphileAmberPreset } from "postgraphile/presets/amber";
import { PostGraphileRelayPreset } from "postgraphile/presets/relay";

export default {
  extends: [PostGraphileAmberPreset, PostGraphileRelayPreset],
};
```

Changes:
- Uses `clientMutationId` in mutations
- Relay-style mutation payloads
- Stricter Node interface compliance

### Lazy JWT Preset

Simple JWT handling:

```js
import { PostGraphileAmberPreset } from "postgraphile/presets/amber";
import { PostGraphileLazyJWTPreset } from "postgraphile/presets/lazy-jwt";

export default {
  extends: [PostGraphileAmberPreset, PostGraphileLazyJWTPreset],
  jwt: {
    secret: process.env.JWT_SECRET,
  },
};
```

## Grafast Engine (V5)

PostGraphile V5 is powered by Grafast, a next-generation GraphQL execution engine. Key improvements:

- **All 4 "epics" completed** (June 2025) — Full feature parity with V4
- **SQL queries built at runtime** — Not plan-time, enabling dynamic optimizations
- **~10% smaller generated SQL** — More efficient queries on average
- **Overhauled polymorphism system** — Native support for interfaces and unions
- **Incremental delivery** — Experimental `@stream` and `@defer` support

### Grafast API Notes

When writing plugins with Grafast steps:

| Old Name | New Name | Usage |
|----------|----------|-------|
| `applyPlan` | `apply` | Apply step modifications |
| `inputPlan` | `baked` | Baked (pre-computed) input handling |

```js
// V5 Release Candidate pattern
const MyFieldPlugin = {
  schema: {
    hooks: {
      GraphQLObjectType_fields_field(field, build, context) {
        return {
          ...field,
          // Use 'apply' instead of 'applyPlan'
          apply($step) {
            return $step.select("column");
          },
        };
      },
    },
  },
};
```

## Configuration Options

### pgServices

Database connection configuration:

```js
pgServices: [
  makePgService({
    // Connection
    connectionString: "postgres://user:pass@host:5432/db",
    // Or individual options:
    // host: "localhost",
    // port: 5432,
    // database: "mydb",
    // user: "postgres",
    // password: "secret",

    // Schemas to expose
    schemas: ["public", "app"],

    // Connection pool
    poolConfig: {
      max: 20,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000,
    },

    // Enable pubsub for subscriptions
    pubsub: true,

    // pgSettings callback
    pgSettings: async (ctx) => ({
      'myapp.user_id': ctx.user?.id?.toString(),
    }),

    // Default role (optional)
    pgDefaultRole: "anonymous",
  }),
],
```

### schema Options

Schema generation settings:

```js
schema: {
  // Default behavior for all entities
  defaultBehavior: "-insert -update -delete",  // Read-only by default

  // Disable introspection in production
  disableIntrospection: process.env.NODE_ENV === "production",

  // JWT type for login mutations
  jwtType: ["public", "jwt_token"],

  // Export schema to file on startup
  exportSchemaSDLPath: "./schema.graphql",

  // Export schema as executable
  exportSchemaIntrospectionResultPath: "./schema.json",

  // Sort schema alphabetically
  sortExport: true,
},
```

### grafast Options

Execution settings:

```js
grafast: {
  // Explain mode for debugging
  explain: process.env.NODE_ENV !== "production",

  // Context values available to resolvers
  context: (ctx) => ({
    user: ctx.user,
    loaders: createLoaders(),
  }),
},
```

### grafserv Options

Server settings:

```js
grafserv: {
  // Port to listen on
  port: 5678,

  // Watch for schema changes
  watch: process.env.NODE_ENV !== "production",

  // Enable GraphiQL
  graphiql: true,

  // Enable WebSockets for subscriptions
  websockets: true,

  // CORS settings
  allowedOrigins: ["https://myapp.com"],

  // Max request size
  maxRequestLength: 100000,
},
```

## Community Plugins

### pg-simplify-inflector

Cleaner naming for relationships:

```bash
npm install @graphile-contrib/pg-simplify-inflector
```

```js
import { PgSimplifyInflectorPlugin } from "@graphile-contrib/pg-simplify-inflector";

export default {
  plugins: [PgSimplifyInflectorPlugin],
};
```

| Default | With Simplify |
|---------|---------------|
| `userByAuthorId` | `author` |
| `postsByAuthorId` | `posts` |
| `allUsers` | `users` |

### pg-many-to-many

Expose many-to-many relationships directly:

```bash
npm install @graphile-contrib/pg-many-to-many
```

```js
import { PgManyToManyPlugin } from "@graphile-contrib/pg-many-to-many";

export default {
  plugins: [PgManyToManyPlugin],
};
```

```sql
-- Junction table
CREATE TABLE post_tags (
  post_id int REFERENCES posts(id),
  tag_id int REFERENCES tags(id),
  PRIMARY KEY (post_id, tag_id)
);
```

```graphql
type Post {
  # Direct access to tags (skipping junction)
  tags: TagsConnection!
}
```

### pg-order-by-related

Order by related table columns:

```bash
npm install @graphile-contrib/pg-order-by-related
```

```js
import { PgOrderByRelatedPlugin } from "@graphile-contrib/pg-order-by-related";

export default {
  plugins: [PgOrderByRelatedPlugin],
};
```

```graphql
query {
  allPosts(orderBy: AUTHOR_BY_AUTHOR_ID__NAME_ASC) {
    nodes { title }
  }
}
```

### postgraphile-plugin-connection-filter

Advanced filtering:

```bash
npm install postgraphile-plugin-connection-filter
```

```js
import ConnectionFilterPlugin from "postgraphile-plugin-connection-filter";

export default {
  plugins: [ConnectionFilterPlugin],
};
```

```graphql
query {
  allPosts(filter: {
    title: { includes: "GraphQL" }
    author: { name: { equalTo: "Alice" } }
    createdAt: { greaterThan: "2024-01-01" }
  }) {
    nodes { id title }
  }
}
```

### pg-pubsub

Subscriptions via LISTEN/NOTIFY:

```bash
npm install @graphile/pg-pubsub
```

```js
import { PgPubsubPlugin } from "@graphile/pg-pubsub";

export default {
  plugins: [PgPubsubPlugin],
  grafserv: {
    websockets: true,
  },
};
```

See [subscriptions.md](subscriptions.md) for usage.

## Writing Custom Plugins

### Basic Plugin Structure

```js
export const MyPlugin = {
  name: "MyPlugin",
  version: "1.0.0",

  // Modify schema building
  schema: {
    hooks: {
      build(build) {
        // Modify build object
        return build;
      },
      init(init, build) {
        // Called at start of schema building
        return init;
      },
      finalize(schema, build) {
        // Final modifications before schema is returned
        return schema;
      },
    },
  },

  // Modify gather phase (introspection)
  gather: {
    hooks: {
      // Modify introspected entities
    },
  },

  // Modify inflection (naming)
  inflection: {
    replace: {
      tableType(previous, options, table) {
        // Custom type naming
        return this.upperCamelCase(table.name);
      },
    },
    add: {
      customInflector(options, entity) {
        return `Custom_${entity.name}`;
      },
    },
  },
};
```

### makeExtendSchemaPlugin

Add custom types and resolvers:

```js
import { makeExtendSchemaPlugin, gql } from "postgraphile/utils";

export const MyExtensionPlugin = makeExtendSchemaPlugin({
  typeDefs: gql`
    type ExternalData {
      id: ID!
      value: String!
    }

    extend type Query {
      externalData(id: ID!): ExternalData
    }

    extend type User {
      externalProfile: ExternalData
    }
  `,
  resolvers: {
    Query: {
      async externalData(_, { id }, context) {
        const response = await fetch(`https://api.example.com/data/${id}`);
        return response.json();
      },
    },
    User: {
      async externalProfile(user, _, context) {
        const response = await fetch(`https://api.example.com/profile/${user.id}`);
        return response.json();
      },
    },
  },
});
```

### makePgSmartTagsPlugin

Apply smart tags programmatically:

```js
import { makePgSmartTagsPlugin } from "postgraphile/utils";

export const SmartTagsPlugin = makePgSmartTagsPlugin([
  {
    kind: "class",
    match: { schemaName: "public", name: "users" },
    tags: { name: "Person" },
  },
  {
    kind: "attribute",
    match: { schemaName: "public", tableName: "users", name: "password_hash" },
    tags: { omit: true },
  },
  {
    kind: "constraint",
    match: { schemaName: "public", tableName: "posts", name: "posts_author_id_fkey" },
    tags: { fieldName: "author", foreignFieldName: "posts" },
  },
]);
```

### makeWrapResolversPlugin

Wrap existing resolvers:

```js
import { makeWrapResolversPlugin } from "postgraphile/utils";

export const LoggingPlugin = makeWrapResolversPlugin({
  // Wrap all Query resolvers
  Query: {
    "*": {
      async resolve(resolver, user, args, context, info) {
        console.log(`Query: ${info.fieldName}`, args);
        const start = Date.now();
        const result = await resolver();
        console.log(`Query: ${info.fieldName} took ${Date.now() - start}ms`);
        return result;
      },
    },
  },
  // Wrap specific mutation
  Mutation: {
    createPost: {
      async resolve(resolver, user, args, context, info) {
        // Validation before resolver
        if (!args.input.title) {
          throw new Error("Title required");
        }
        return resolver();
      },
    },
  },
});
```

## Plugin Order

Plugins are applied in array order. Put dependencies first:

```js
plugins: [
  // Foundation plugins first
  PgSimplifyInflectorPlugin,

  // Feature plugins
  PgManyToManyPlugin,
  ConnectionFilterPlugin,

  // Custom plugins last
  MyCustomPlugin,
],
```

## Running PostGraphile

### CLI

```bash
npx postgraphile -c postgres://localhost/mydb --schema public --watch
```

### Library (Express)

```js
import express from "express";
import { postgraphile } from "postgraphile";
import preset from "./graphile.config.mjs";

const app = express();
app.use(postgraphile(preset));
app.listen(5678);
```

### Library (Standalone)

```js
import { createServer } from "node:http";
import { grafserv } from "postgraphile/grafserv/node";
import preset from "./graphile.config.mjs";

const serv = grafserv({ preset });
const server = createServer(serv.createHandler());
server.listen(5678);
```

## Environment-Specific Configuration

```js
// graphile.config.mjs
const isDev = process.env.NODE_ENV !== "production";

export default {
  extends: [PostGraphileAmberPreset],

  schema: {
    disableIntrospection: !isDev,
    exportSchemaSDLPath: isDev ? "./schema.graphql" : undefined,
  },

  grafast: {
    explain: isDev,
  },

  grafserv: {
    watch: isDev,
    graphiql: isDev,
    port: parseInt(process.env.PORT || "5678"),
  },
};
```

## Debugging

### Schema Export

```js
schema: {
  exportSchemaSDLPath: "./schema.graphql",
},
```

### Explain Mode

```js
grafast: {
  explain: true,  // Shows execution plan in response
},
```

### Verbose Logging

```bash
DEBUG=postgraphile:* npx postgraphile ...
```

## Next Steps

- [SKILL.md](SKILL.md) — Quick reference
- [security.md](security.md) — Production security setup
