# PostgreSQL → GraphQL Schema Translation

PostGraphile automatically generates a GraphQL schema from your PostgreSQL database. Understanding the translation rules is essential for designing your database schema to produce the desired GraphQL API.

## Core Inflection Rules

### Table Names → Types

Tables become GraphQL types with UpperCamelCase names:

| PostgreSQL Table | GraphQL Type |
|------------------|--------------|
| `users` | `User` |
| `blog_posts` | `BlogPost` |
| `pending_users` | `PendingUser` |
| `user_profile` | `UserProfile` |

### Column Names → Fields

Columns become GraphQL fields with camelCase names:

| PostgreSQL Column | GraphQL Field |
|-------------------|---------------|
| `id` | `id` |
| `created_at` | `createdAt` |
| `user_id` | `userId` |
| `is_active` | `isActive` |
| `email_address` | `emailAddress` |

### Generated Query Fields

For each table, PostGraphile generates:

```graphql
type Query {
  # Single row by primary key
  user(id: Int!): User

  # Single row by unique constraint
  userByEmail(email: String!): User

  # All rows with pagination
  allUsers(
    first: Int
    last: Int
    after: Cursor
    before: Cursor
    condition: UserCondition
    filter: UserFilter
    orderBy: [UsersOrderBy!]
  ): UsersConnection
}
```

### Generated Mutation Fields

For each table (unless omitted):

```graphql
type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload
  updateUser(input: UpdateUserInput!): UpdateUserPayload
  updateUserByEmail(input: UpdateUserByEmailInput!): UpdateUserPayload
  deleteUser(input: DeleteUserInput!): DeleteUserPayload
  deleteUserByEmail(input: DeleteUserByEmailInput!): DeleteUserPayload
}
```

## Foreign Key Relationships

### Forward Relation (Many-to-One)

A foreign key creates a field on the referencing type:

```sql
CREATE TABLE posts (
  id serial PRIMARY KEY,
  author_id int REFERENCES users(id),
  title text
);
```

```graphql
type Post {
  id: Int!
  authorId: Int
  title: String

  # Forward relation - fetch the author
  userByAuthorId: User
}
```

### Reverse Relation (One-to-Many)

The referenced table gets a connection field:

```graphql
type User {
  id: Int!
  name: String

  # Reverse relation - fetch posts by this user
  postsByAuthorId(
    first: Int
    after: Cursor
    # ... pagination args
  ): PostsConnection!
}
```

### Naming with pg-simplify-inflector

The `@graphile-contrib/pg-simplify-inflector` plugin produces cleaner names:

| Default | With Simplify Inflector |
|---------|-------------------------|
| `userByAuthorId` | `author` |
| `postsByAuthorId` | `posts` |
| `allUsers` | `users` |
| `userByNodeId` | `user` |

## Type Mappings

### Scalar Types

| PostgreSQL Type | GraphQL Type |
|-----------------|--------------|
| `integer`, `int4` | `Int` |
| `bigint`, `int8` | `BigInt` (custom scalar) |
| `smallint`, `int2` | `Int` |
| `real`, `float4` | `Float` |
| `double precision`, `float8` | `Float` |
| `numeric`, `decimal` | `BigFloat` (custom scalar) |
| `boolean`, `bool` | `Boolean` |
| `text`, `varchar`, `char` | `String` |
| `uuid` | `UUID` (custom scalar) |
| `json` | `JSON` (custom scalar) |
| `jsonb` | `JSON` (custom scalar) |
| `timestamp` | `Datetime` (custom scalar) |
| `timestamptz` | `Datetime` (custom scalar) |
| `date` | `Date` (custom scalar) |
| `time` | `Time` (custom scalar) |
| `interval` | `Interval` (custom scalar) |

### Array Types

PostgreSQL arrays become GraphQL lists:

```sql
CREATE TABLE posts (
  id serial PRIMARY KEY,
  tags text[]
);
```

```graphql
type Post {
  id: Int!
  tags: [String]
}
```

### Enum Types

PostgreSQL enums become GraphQL enums:

```sql
CREATE TYPE post_status AS ENUM ('draft', 'published', 'archived');

CREATE TABLE posts (
  id serial PRIMARY KEY,
  status post_status DEFAULT 'draft'
);
```

```graphql
enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

type Post {
  id: Int!
  status: PostStatus
}
```

### Composite Types

PostgreSQL composite types become GraphQL types:

```sql
CREATE TYPE address AS (
  street text,
  city text,
  country text,
  postal_code text
);

CREATE TABLE users (
  id serial PRIMARY KEY,
  home_address address
);
```

```graphql
type Address {
  street: String
  city: String
  country: String
  postalCode: String
}

type User {
  id: Int!
  homeAddress: Address
}
```

### Domain Types

Domains inherit the GraphQL type of their base:

```sql
CREATE DOMAIN email AS text CHECK (value ~ '@');
```

The `email` domain maps to `String` in GraphQL.

## Connection Types (Relay Pagination)

Tables and `SETOF` functions generate Relay-style connections:

```graphql
type UsersConnection {
  nodes: [User!]!
  edges: [UsersEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UsersEdge {
  node: User!
  cursor: Cursor!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: Cursor
  endCursor: Cursor
}
```

## Custom Inflection

### Using Smart Tags

Override names with `@name`:

```sql
COMMENT ON TABLE users IS '@name Person';
COMMENT ON COLUMN users.email_address IS '@name email';
```

```graphql
type Person {
  email: String  # was emailAddress
}
```

### Custom Inflection Plugins

For systematic changes, create an inflection plugin:

```js
// my-inflection-plugin.js
export const MyInflectionPlugin = {
  inflection: {
    replace: {
      // Remove "ByNodeId" suffix
      tableType(previous, options, table) {
        return this.upperCamelCase(table.name);
      },
    },
  },
};
```

## Reserved Words and Conflicts

### PostgreSQL Reserved Words

These cause issues and should be avoided or renamed:

| Reserved Word | Problem | Solution |
|---------------|---------|----------|
| `user` | PostgreSQL keyword | Use `users` or `@name` tag |
| `order` | PostgreSQL keyword | Use `orders` or `@name` tag |
| `group` | PostgreSQL keyword | Use `groups` or `@name` tag |
| `select`, `from`, `where` | SQL keywords | Avoid or rename |

### GraphQL Conflicts

| Conflict | Problem | Solution |
|----------|---------|----------|
| `id` with Node interface | Conflicts with global ID | PostGraphile handles this |
| `query`, `mutation` | GraphQL root types | Rename with `@name` |
| Starting with numbers | Invalid GraphQL | Tables can't start with numbers |

## Schema-Level Control

### Multiple Schemas

Expose specific schemas:

```js
// graphile.config.mjs
export default {
  pgServices: [makePgService({
    connectionString: process.env.DATABASE_URL,
    schemas: ["public", "app"],  // Expose both schemas
  })],
};
```

With multiple schemas, types may be prefixed to avoid conflicts.

### Excluding Tables

In V5, use `@behavior` to hide tables (note: `@omit` only works with V4 preset):

```sql
-- V5 (recommended)
COMMENT ON TABLE internal_logs IS '@behavior -*';

-- V4 preset only
COMMENT ON TABLE internal_logs IS '@omit';
```

## Views

Views are treated like tables:

```sql
CREATE VIEW active_users AS
  SELECT * FROM users WHERE is_active = true;
```

```graphql
type ActiveUser {
  id: Int!
  name: String
  # ... all columns from view
}

type Query {
  activeUser(id: Int!): ActiveUser
  allActiveUsers: ActiveUsersConnection
}
```

For mutations on views, you need:
1. Updatable views (simple views)
2. `INSTEAD OF` triggers (complex views)
3. `@primaryKey` smart tag (if no PK detected)

```sql
COMMENT ON VIEW active_users IS '@primaryKey id';
```

## Nullability

### Column Nullability

| PostgreSQL | GraphQL |
|------------|---------|
| `NOT NULL` | Non-nullable `Type!` |
| Nullable (default) | Nullable `Type` |

```sql
CREATE TABLE users (
  id serial PRIMARY KEY,        -- Int! (NOT NULL)
  email text NOT NULL,          -- String!
  nickname text                 -- String (nullable)
);
```

```graphql
type User {
  id: Int!
  email: String!
  nickname: String
}
```

### Function Return Nullability

Functions returning `SETOF` are always connections. Functions returning single values respect `RETURNS ... NOT NULL`:

```sql
-- Nullable return (default)
CREATE FUNCTION maybe_user(id int) RETURNS users AS $$ ... $$;

-- Non-null return
CREATE FUNCTION guaranteed_user(id int) RETURNS users NOT NULL AS $$ ... $$;
```

## Polymorphism (V5)

PostGraphile V5 supports GraphQL interfaces and unions, enabling polymorphic patterns.

### Interfaces from Composite Types

```sql
CREATE TYPE media_item AS (
  id int,
  title text,
  created_at timestamptz
);

COMMENT ON TYPE media_item IS '@interface mode:union';
```

Tables returning this type implement the interface automatically.

### Interfaces from Relational Tables

```sql
-- Base table defines shared columns
CREATE TABLE content (
  id serial PRIMARY KEY,
  content_type text NOT NULL,  -- discriminator column
  title text NOT NULL,
  created_at timestamptz DEFAULT now()
);

COMMENT ON TABLE content IS '@interface mode:relational type:Content';

-- Child tables extend the base
CREATE TABLE articles (
  id int PRIMARY KEY REFERENCES content(id),
  body text
);

CREATE TABLE videos (
  id int PRIMARY KEY REFERENCES content(id),
  url text NOT NULL,
  duration int
);

COMMENT ON TABLE articles IS '@implements Content';
COMMENT ON TABLE videos IS '@implements Content';
```

Produces:
```graphql
interface Content {
  id: Int!
  title: String!
  createdAt: Datetime!
}

type Article implements Content {
  id: Int!
  title: String!
  body: String
  createdAt: Datetime!
}

type Video implements Content {
  id: Int!
  title: String!
  url: String!
  duration: Int
  createdAt: Datetime!
}
```

### Unions

Group unrelated types into unions:

```sql
COMMENT ON TABLE articles IS '@unionMember SearchResult';
COMMENT ON TABLE users IS '@unionMember SearchResult';
COMMENT ON TABLE products IS '@unionMember SearchResult';
```

```graphql
union SearchResult = Article | User | Product
```

### Requirements

- **Primary keys required** — All polymorphic tables must have primary keys
- **Discriminator column** — Relational interfaces need a way to determine type
- **Compatible columns** — Implementing types must have matching columns for interface fields

See [smart-tags.md](smart-tags.md) for all polymorphism tags (`@interface`, `@implements`, `@unionMember`, `@ref`).

## Incremental Delivery (Experimental)

V5 supports GraphQL incremental delivery with `@stream` and `@defer` directives.

### @defer — Defer Non-Critical Fields

Defer expensive fields to load after initial response:

```graphql
query {
  user(id: 1) {
    id
    name
    ... @defer {
      expensiveAnalytics {
        totalPosts
        totalComments
        engagementScore
      }
    }
  }
}
```

Response streams in chunks:
1. Initial: `{ id, name }`
2. Deferred: `{ expensiveAnalytics: { ... } }`

### @stream — Stream List Items

Stream large lists progressively:

```graphql
query {
  allPosts(first: 1000) @stream(initialCount: 10) {
    nodes {
      id
      title
    }
  }
}
```

Response:
1. Initial: First 10 items
2. Streamed: Remaining items in batches

### Enabling Incremental Delivery

```js
// graphile.config.mjs
export default {
  grafast: {
    experimentalIncrementalDelivery: true,
  },
};
```

### Use Cases

- **Large exports** — Stream CSV/JSON exports without timeouts
- **Dashboard loading** — Show critical data first, analytics later
- **Search results** — Display initial results immediately, load more progressively

> **Note:** Incremental delivery requires client support (e.g., `graphql-ws` with proper handling).

## Next Steps

- [smart-tags.md](smart-tags.md) — Customize names, hide fields, control behaviors
- [functions.md](functions.md) — Add computed columns and custom operations
