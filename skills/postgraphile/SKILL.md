---
name: postgraphile
description: >
  PostGraphile V5 (Release Candidate) patterns for generating GraphQL APIs from PostgreSQL.
  Covers schema translation, smart tags, behavior system, computed columns, custom mutations,
  RLS security, subscriptions via LISTEN/NOTIFY, and polymorphism (interfaces/unions).
  Assumes knowledge of SQL, PostgreSQL, and GraphQL fundamentals.
  Use when building or customizing a PostGraphile API.
---

# PostGraphile Quick Reference

PostGraphile auto-generates a GraphQL API from your PostgreSQL schema. This skill covers how to control that generation.

## How Does X Translate to GraphQL?

| PostgreSQL Entity | GraphQL Result | Example |
|-------------------|----------------|---------|
| Table `users` | Type `User`, queries `user`, `allUsers` | `type User { ... }` |
| Column `created_at` | Field `createdAt` (camelCase) | `createdAt: Datetime` |
| Foreign key `author_id → users` | Relation field `userByAuthorId` | `userByAuthorId: User` |
| Reverse FK (users has many posts) | Connection field `postsByAuthorId` | `postsByAuthorId: PostsConnection` |
| `SETOF` function return | Connection with pagination | `allActiveUsers: UsersConnection` |
| Single row function return | Single object | `currentUser: User` |
| `STABLE`/`IMMUTABLE` function | Query field | `query { myFunc }` |
| `VOLATILE` function | Mutation field | `mutation { doThing }` |
| Enum type | GraphQL enum | `enum Status { ACTIVE, INACTIVE }` |
| Composite type | GraphQL type | `type Address { ... }` |

See [schema.md](schema.md) for full translation rules.

## How Do I Customize X?

| Goal | Method | Example |
|------|--------|---------|
| Rename field/type | `@name` smart tag | `COMMENT ON TABLE users IS '@name Person';` |
| Hide field completely | `@behavior -* ` tag | `@behavior -select` on column |
| Hide specific operations | `@behavior` with operations | `@behavior -insert -update` |
| Rename relation field | `@fieldName` / `@foreignFieldName` | `@fieldName author` on FK |
| Add computed field | Create function `tablename_fieldname(row)` | `users_full_name(users)` |
| Add custom query | Create `STABLE` function | `CREATE FUNCTION search(...)` |
| Add custom mutation | Create `VOLATILE` function | `CREATE FUNCTION do_action(...)` |
| Use simpler names | Install `pg-simplify-inflector` | `author` instead of `userByAuthorId` |
| Define interface | `@interface` tag | `@interface mode:relational` on base table |
| Implement interface | `@implements` tag | `@implements Content` on child table |

See [smart-tags.md](smart-tags.md) for all customization options.

## Function Volatility → Placement

| Volatility | GraphQL Location | Use For |
|------------|------------------|---------|
| `IMMUTABLE` | Query | Pure computations |
| `STABLE` | Query | Reads from DB, no side effects |
| `VOLATILE` | Mutation | Writes, side effects |

```sql
-- Query (STABLE reads current state)
CREATE FUNCTION search_users(query text) RETURNS SETOF users
LANGUAGE sql STABLE AS $$
  SELECT * FROM users WHERE name ILIKE '%' || query || '%';
$$;

-- Mutation (VOLATILE has side effects)
CREATE FUNCTION register_user(email text, name text) RETURNS users
LANGUAGE plpgsql VOLATILE AS $$
DECLARE new_user users;
BEGIN
  INSERT INTO users (email, name) VALUES (email, name) RETURNING * INTO new_user;
  RETURN new_user;
END;
$$;
```

See [functions.md](functions.md) for computed columns and custom resolvers.

## Quick Smart Tag Reference

| Tag | Purpose | Example |
|-----|---------|---------|
| `@name newName` | Rename entity | `@name Person` |
| `@behavior -* ` | Hide completely | `@behavior -*` |
| `@behavior -ops` | Hide operations | `@behavior -insert -update -delete` |
| `@deprecated reason` | Mark deprecated | `@deprecated Use emailAddress instead` |
| `@fieldName name` | Rename FK relation | `@fieldName author` |
| `@foreignFieldName name` | Rename reverse relation | `@foreignFieldName writtenPosts` |
| `@behavior +modifier` | Add behavior | `@behavior +filterBy` |
| `@behavior -modifier` | Remove behavior | `@behavior -update -delete` |
| `@interface mode:X` | Define interface | `@interface mode:relational` |
| `@implements Name` | Implement interface | `@implements Content` |
| `@unionMember Name` | Join union | `@unionMember SearchResult` |

Tags can be applied via:
1. SQL `COMMENT` statements
2. `postgraphile.tags.json5` file (requires `TagsFilePlugin`)
3. `makePgSmartTagsPlugin` in code

See [smart-tags.md](smart-tags.md) for complete reference.

## Behavior System (V5)

V5 uses behaviors for fine-grained control over what gets exposed.

```sql
-- Only allow select and filter, no mutations
COMMENT ON TABLE audit_logs IS '@behavior +select +filter -insert -update -delete';
```

Common behaviors:
- `+select` / `-select` — Can read rows
- `+insert` / `-insert` — Can insert rows
- `+update` / `-update` — Can update rows
- `+delete` / `-delete` — Can delete rows
- `+filterBy` / `-filterBy` — Can filter by this column
- `+orderBy` / `-orderBy` — Can order by this column
- `+connection` / `-connection` — Include in connections

See [smart-tags.md](smart-tags.md) for behavior system details.

## Security Pattern (RLS + pgSettings)

PostGraphile passes JWT claims to PostgreSQL via `pgSettings`, enabling RLS:

```sql
-- Helper function to get current user
CREATE FUNCTION current_user_id() RETURNS int AS $$
  SELECT nullif(current_setting('myapp.user_id', true), '')::int
$$ LANGUAGE sql STABLE;

-- RLS policy using the helper
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;
CREATE POLICY posts_owner ON posts
  USING (author_id = current_user_id());
```

Configure in PostGraphile:
```js
// graphile.config.mjs
export default {
  pgServices: [makePgService({
    pgSettings: async (req) => ({
      'myapp.user_id': req.user?.id?.toString(),
      'myapp.role': req.user?.role,
    }),
  })],
};
```

See [security.md](security.md) for JWT integration and RLS patterns.

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| `VOLATILE` for read-only function | Appears as mutation | Use `STABLE` for queries |
| No named arguments | Arguments named `arg0`, `arg1` | Use named parameters |
| Missing `SETOF` for lists | Returns single item | Use `SETOF tablename` for connections |
| Forgetting `SECURITY DEFINER` risk | Function runs as owner | Use `SECURITY INVOKER` or be deliberate |
| RLS without `current_setting` helper | Repeating `current_setting()` everywhere | Create helper function |
| Not using connection pooling | Too many DB connections | Use pgBouncer or built-in pool |
| Using `@omit` without V4 preset | `@omit` not working in V5 | Use `@behavior -* ` instead |
| Ignoring V5 behavior defaults | Unexpected exposure | Set `schema.defaultBehavior` in config |
| Missing `TagsFilePlugin` | Tags file not loaded | Add `TagsFilePlugin` to plugins array |
| Missing primary keys for polymorphism | Interface/union errors | Ensure all polymorphic tables have PKs |
| Custom function returns row, not edge | Relay `@appendEdge` won't work | Use CRUD mutations or `makeExtendSchemaPlugin` |
| Returning plain JS objects in plans | Null values, silent failures | Use `object()` and `constant()` from grafast |

## Subscriptions Quick Reference

PostGraphile uses PostgreSQL `LISTEN/NOTIFY` for subscriptions:

```sql
-- Trigger to notify on changes
CREATE FUNCTION notify_post_change() RETURNS trigger AS $$
BEGIN
  PERFORM pg_notify('postgraphile:post', json_build_object('id', NEW.id)::text);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER post_changed AFTER INSERT OR UPDATE ON posts
  FOR EACH ROW EXECUTE FUNCTION notify_post_change();
```

> **CRITICAL: Grafast Step Requirement**
>
> In subscription plugins, you cannot return plain JavaScript objects. Use `object()` and `constant()` from `postgraphile/grafast`:
> ```js
> // WRONG: return { node: $node, cursor: null };
> // CORRECT:
> import { object, constant } from "postgraphile/grafast";
> return object({ node: $node, cursor: constant(null) });
> ```

See [subscriptions.md](subscriptions.md) for setup and custom subscriptions.

## Configuration Quick Reference

```js
// graphile.config.mjs
import { PostGraphileAmberPreset } from "postgraphile/presets/amber";
import { makePgService } from "postgraphile/adaptors/pg";

export default {
  extends: [PostGraphileAmberPreset],
  pgServices: [makePgService({
    connectionString: process.env.DATABASE_URL,
    schemas: ["public"],
  })],
  schema: {
    defaultBehavior: "-insert -update -delete", // Read-only by default
  },
  grafserv: {
    port: 5678,
    watch: true,
  },
};
```

See [plugins.md](plugins.md) for presets, plugins, and advanced configuration.

## Supporting Files

| Topic | File | Covers |
|-------|------|--------|
| Schema Translation | [schema.md](schema.md) | PostgreSQL → GraphQL naming, types, polymorphism |
| Smart Tags | [smart-tags.md](smart-tags.md) | @name, @behavior, polymorphism, relationship tags |
| Functions | [functions.md](functions.md) | Computed columns, custom queries/mutations |
| Security | [security.md](security.md) | RLS, pgSettings, JWT integration |
| Subscriptions | [subscriptions.md](subscriptions.md) | LISTEN/NOTIFY, realtime updates, pgSubscriber |
| Plugins | [plugins.md](plugins.md) | Configuration, presets, Grafast, community plugins |

## Related Skills

PostGraphile builds on:
- **sql** skill: Query patterns, JOINs, CTEs, transactions
- **postgres** skill: RLS, JSONB, functions, types, permissions
- **graphql** skill (relay-graphql plugin): Types, operations, nullability
- **relay** skill (relay-graphql plugin): Connections, pagination, fragments
