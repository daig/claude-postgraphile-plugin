# Smart Tags and Behavior System

Smart tags customize how PostGraphile exposes your database schema. They can rename entities, hide fields, modify relationships, and control GraphQL generation behavior.

## Applying Smart Tags

### Method 1: SQL Comments

```sql
COMMENT ON TABLE users IS '@name Person
@deprecated Use people table instead';

COMMENT ON COLUMN users.email IS '@name emailAddress';

COMMENT ON CONSTRAINT users_org_id_fkey ON users IS '@fieldName organization';
```

Multiple tags on one entity use newlines.

### Method 2: Tags File (postgraphile.tags.json5)

> **Note:** In V5, you must explicitly add `TagsFilePlugin` for the tags file to be loaded:
>
> ```js
> import { TagsFilePlugin } from "postgraphile/presets/tags-file";
> export default { plugins: [TagsFilePlugin] };
> ```

```json5
{
  version: 1,
  config: {
    class: {
      "public.users": {
        tags: { name: "Person" },
        description: "A person using the app"
      },
      "public.internal_logs": {
        tags: { behavior: "-*" }  // Hide completely in V5
      }
    },
    attribute: {
      "public.users.email_address": {
        tags: { name: "email" }
      }
    },
    constraint: {
      "public.posts.posts_author_id_fkey": {
        tags: {
          fieldName: "author",
          foreignFieldName: "posts"
        }
      }
    }
  }
}
```

### Method 3: Plugin (makePgSmartTagsPlugin)

```js
import { makePgSmartTagsPlugin } from "postgraphile/utils";

export const MyTagsPlugin = makePgSmartTagsPlugin({
  kind: "class",
  match: { schemaName: "public", name: "users" },
  tags: { name: "Person" },
});
```

## Core Smart Tags

### @name — Rename Anything

```sql
-- Rename table → type
COMMENT ON TABLE blog_posts IS '@name Article';

-- Rename column → field
COMMENT ON COLUMN users.email_address IS '@name email';

-- Rename function → query/mutation
COMMENT ON FUNCTION search_users IS '@name searchPeople';

-- Rename enum value
COMMENT ON TYPE post_status IS '@enumName draft UNPUBLISHED';
```

### @omit — Hide from Schema (V4 Preset Only)

> **V5 Note:** `@omit` only works when using the V4 compatibility preset. In V5, use `@behavior -* ` to hide entities. See the [Behavior System](#behavior-system-v5) section below.

Hide completely (V4 preset):
```sql
COMMENT ON TABLE internal_logs IS '@omit';
COMMENT ON COLUMN users.password_hash IS '@omit';
```

**V5 equivalent:**
```sql
COMMENT ON TABLE internal_logs IS '@behavior -*';
COMMENT ON COLUMN users.password_hash IS '@behavior -select';
```

Hide specific operations (V4 preset):
```sql
-- Hide mutations but allow queries
COMMENT ON TABLE audit_events IS '@omit create,update,delete';

-- Hide from specific locations
COMMENT ON COLUMN users.created_at IS '@omit create,update';
```

**V5 equivalent:**
```sql
COMMENT ON TABLE audit_events IS '@behavior -insert -update -delete';
COMMENT ON COLUMN users.created_at IS '@behavior -insert -update';
```

Omit flags (V4 preset only):
| V4 Flag | V5 Behavior Equivalent |
|---------|----------------------|
| `create` | `-insert` |
| `update` | `-update` |
| `delete` | `-delete` |
| `all` | `-*` |
| `many` | `-connection -list` |
| `execute` | `-queryField -mutationField` |
| `filter` | `-filterBy` |
| `order` | `-orderBy` |

### @deprecated — Mark Deprecated

```sql
COMMENT ON COLUMN users.email IS '@deprecated Use emailAddress instead';
```

Produces:
```graphql
type User {
  email: String @deprecated(reason: "Use emailAddress instead")
}
```

## Relationship Tags

### @fieldName — Rename Forward Relation

```sql
-- posts.author_id → users
-- Default: userByAuthorId
-- With tag: author
COMMENT ON CONSTRAINT posts_author_id_fkey ON posts IS '@fieldName author';
```

### @foreignFieldName — Rename Reverse Relation

```sql
-- On User type, the posts connection
-- Default: postsByAuthorId
-- With tag: writtenPosts
COMMENT ON CONSTRAINT posts_author_id_fkey ON posts IS
  '@fieldName author
   @foreignFieldName writtenPosts';
```

### @foreignKey — Virtual Foreign Key

For views or tables without actual FKs:

```sql
COMMENT ON VIEW user_summary IS
  '@foreignKey (user_id) REFERENCES users (id)';
```

Multiple virtual FKs:
```sql
COMMENT ON VIEW post_details IS
  '@foreignKey (author_id) REFERENCES users (id)|@fieldName author
   @foreignKey (category_id) REFERENCES categories (id)|@fieldName category';
```

### @primaryKey — Virtual Primary Key

For views without detectable PKs:

```sql
COMMENT ON VIEW active_users IS '@primaryKey id';
-- Composite: '@primaryKey tenant_id,user_id'
```

### @unique — Virtual Unique Constraint

Enables `userByFieldname` queries:

```sql
COMMENT ON VIEW user_summary IS '@unique email';
```

## Function Tags

### @arg0variant, @arg1variant, etc.

Control argument type variants:

```sql
COMMENT ON FUNCTION update_user(int, json) IS
  '@arg1variant patch';  -- Use patch input instead of full type
```

### @resultFieldName

Name the field in mutation payload:

```sql
COMMENT ON FUNCTION register_user(text, text) IS
  '@resultFieldName newUser';
```

Produces:
```graphql
type RegisterUserPayload {
  newUser: User  # instead of default 'user'
}
```

### @returnType

Override inferred return type:

```sql
COMMENT ON FUNCTION search_users(text) IS '@returnType [User!]!';
```

### @sortable

Enable sorting on function results:

```sql
COMMENT ON FUNCTION get_popular_posts() IS '@sortable';
```

## Behavior System (V5)

PostGraphile V5 uses behaviors for fine-grained control. Behaviors are more powerful than `@omit`.

### Syntax

```sql
-- Add behaviors with +
COMMENT ON TABLE users IS '@behavior +select +filter';

-- Remove behaviors with -
COMMENT ON TABLE audit_logs IS '@behavior -insert -update -delete';

-- Combine
COMMENT ON TABLE users IS '@behavior +select +insert -update -delete';
```

### Table/View Behaviors

| Behavior | Controls |
|----------|----------|
| `+select` / `-select` | Can query rows |
| `+insert` / `-insert` | Create mutation |
| `+update` / `-update` | Update mutations |
| `+delete` / `-delete` | Delete mutations |
| `+single` / `-single` | `user(id: ...)` query |
| `+list` / `-list` | Include in connections |
| `+connection` / `-connection` | Relay connection |

### Column Behaviors

| Behavior | Controls |
|----------|----------|
| `+select` / `-select` | Include in type |
| `+filterBy` / `-filterBy` | Can filter by column |
| `+orderBy` / `-orderBy` | Can order by column |
| `+insert` / `-insert` | Can set on create |
| `+update` / `-update` | Can modify on update |

### Function Behaviors

| Behavior | Controls |
|----------|----------|
| `+queryField` / `-queryField` | Root query field |
| `+mutationField` / `-mutationField` | Root mutation field |
| `+typeField` / `-typeField` | Computed column |

### Default Behaviors

Set global defaults in config:

```js
// graphile.config.mjs
export default {
  schema: {
    defaultBehavior: "-insert -update -delete",  // Read-only by default
  },
};
```

Then opt-in per table:
```sql
COMMENT ON TABLE posts IS '@behavior +insert +update';
```

### Behavior Inheritance

Behaviors cascade: global defaults → schema → table → column.

More specific wins, or use `!` to force:

```sql
-- Force even if global default says otherwise
COMMENT ON TABLE users IS '@behavior !+insert';
```

## Polymorphism Tags (V5)

PostGraphile V5 supports GraphQL interfaces and unions via smart tags. There are three interface modes depending on your database design.

### @interface mode:single — Single Table Inheritance

All polymorphic types in one table with a discriminator column:

```sql
CREATE TABLE items (
  id serial PRIMARY KEY,
  type text NOT NULL,  -- discriminator: 'TOPIC', 'POST', 'DIVIDER'
  title text,
  body text,           -- POST only
  color text           -- DIVIDER only
);

COMMENT ON TABLE items IS E'@interface mode:single type:type
@type TOPIC name:Topic
@type POST name:Post attributes:body
@type DIVIDER name:Divider attributes:color';
```

### @interface mode:relational — Joined Table Inheritance

Common fields in parent table, type-specific fields in child tables:

```sql
CREATE TABLE content (
  id serial PRIMARY KEY,
  type text NOT NULL,
  title text NOT NULL,
  created_at timestamptz DEFAULT now()
);

CREATE TABLE articles (
  id int PRIMARY KEY REFERENCES content(id),
  body text
);

CREATE TABLE videos (
  id int PRIMARY KEY REFERENCES content(id),
  url text NOT NULL,
  duration int
);

COMMENT ON TABLE content IS E'@interface mode:relational type:type
@type ARTICLE references:articles
@type VIDEO references:videos';
```

### @interface mode:union — Independent Tables

Independent tables sharing fields via a composite type (no shared parent table):

```sql
CREATE TYPE media_item AS (
  id int,
  title text,
  created_at timestamptz
);

COMMENT ON TYPE media_item IS '@interface mode:union';

-- Each table implements independently
COMMENT ON TABLE articles IS '@implements MediaItem';
COMMENT ON TABLE videos IS '@implements MediaItem';
```

> **Note:** Cannot return `mode:union` types from PostgreSQL functions; use `extendSchema` instead.

### @implements — Table Implements Interface

Mark tables as implementing an interface:

```sql
CREATE TABLE articles (
  id serial PRIMARY KEY,
  title text NOT NULL,
  body text,
  created_at timestamptz DEFAULT now()
);

CREATE TABLE videos (
  id serial PRIMARY KEY,
  title text NOT NULL,
  url text NOT NULL,
  duration int,
  created_at timestamptz DEFAULT now()
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

### @unionMember — GraphQL Union Membership

Add types to a GraphQL union:

```sql
-- Define union via smart tag
COMMENT ON TABLE articles IS '@unionMember SearchResult';
COMMENT ON TABLE users IS '@unionMember SearchResult';
COMMENT ON TABLE comments IS '@unionMember SearchResult';
```

Produces:
```graphql
union SearchResult = Article | User | Comment
```

### Polymorphism Requirements

For polymorphism to work correctly:
1. **Tables need primary keys** — Required for type resolution
2. **Shared columns must match** — For interfaces, implementing tables must have compatible columns
3. **Use refs for cross-type linking** — When relating polymorphic types

### @ref — Cross-Type References

Link to polymorphic types:

```sql
COMMENT ON TABLE comments IS
  '@ref commentable via:(commentable_type, commentable_id)
   @refVia commentable articles:Article videos:Video';
```

## Complex Examples

### Read-Only Public Data

```sql
-- Global: read-only
-- In config: defaultBehavior: "-insert -update -delete"

-- Allow insert on users (registration)
COMMENT ON TABLE users IS '@behavior +insert';

-- Allow update on user's own profile
COMMENT ON TABLE user_profiles IS '@behavior +update';

-- Completely hide internal table
COMMENT ON TABLE internal_metrics IS '@behavior -*';
```

### Multi-Tenant with Hidden Tenant Column

```sql
-- Hide tenant_id from GraphQL but use in RLS
COMMENT ON COLUMN posts.tenant_id IS '@behavior -select';

-- Or just hide from input (visible in queries, not in mutations)
COMMENT ON COLUMN posts.tenant_id IS '@behavior -insert -update';
```

### Simplified Relationship Names

Using tags file for cleaner FK names:

```json5
{
  version: 1,
  config: {
    constraint: {
      "public.posts.posts_author_id_fkey": {
        tags: { fieldName: "author", foreignFieldName: "posts" }
      },
      "public.comments.comments_post_id_fkey": {
        tags: { fieldName: "post", foreignFieldName: "comments" }
      },
      "public.comments.comments_author_id_fkey": {
        tags: { fieldName: "author", foreignFieldName: "comments" }
      }
    }
  }
}
```

### Deprecated Fields During Migration

```sql
-- Old column (deprecated)
COMMENT ON COLUMN users.email IS
  '@deprecated Use emailAddress instead
   @name oldEmail';

-- New column
COMMENT ON COLUMN users.email_address IS '@name email';
```

## Debugging Smart Tags

### Check Applied Tags

In V5, use the `--export-schema-sdl` flag:

```bash
postgraphile --export-schema-sdl schema.graphql
```

Inspect the generated `schema.graphql` to verify tags applied correctly.

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Tag not applying | Wrong entity name | Check schema.tablename format |
| Relation name wrong | Tagged wrong constraint | Find correct FK constraint name |
| Behavior not working | Missing `+`/`-` prefix or wrong preset | Ensure V5 preset, check `+`/`-` prefix |
| `@omit` not working | Using V5 without V4 preset | Use `@behavior -* ` in V5 instead |
| Tags file not loading | Missing TagsFilePlugin | Add `TagsFilePlugin` to plugins |

### Find Constraint Names

```sql
SELECT conname
FROM pg_constraint
WHERE conrelid = 'posts'::regclass
  AND contype = 'f';
```

## Next Steps

- [functions.md](functions.md) — Add computed columns and custom queries
- [schema.md](schema.md) — Understand auto-generated names
