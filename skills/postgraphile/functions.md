# Functions: Computed Columns, Custom Queries, and Mutations

PostgreSQL functions extend your GraphQL API beyond basic CRUD. PostGraphile automatically exposes functions based on their signature and volatility.

## Computed Columns

Computed columns add fields to existing types without modifying the table.

### Naming Convention

```sql
CREATE FUNCTION tablename_fieldname(row tablename) RETURNS type AS $$ ... $$;
```

The function name pattern `tablename_fieldname` with first argument of `tablename` type creates a field on that type.

### Basic Example

```sql
CREATE FUNCTION users_full_name(u users) RETURNS text AS $$
  SELECT u.first_name || ' ' || u.last_name
$$ LANGUAGE sql STABLE;
```

```graphql
type User {
  id: Int!
  firstName: String
  lastName: String
  fullName: String  # Computed from function
}
```

### Volatility Requirements

Computed columns **must** be `STABLE` or `IMMUTABLE`:

| Volatility | Meaning | Use For |
|------------|---------|---------|
| `IMMUTABLE` | Same input always returns same output | Pure computations |
| `STABLE` | Returns same result within transaction | Reads from DB |
| `VOLATILE` | May return different results | **Not for computed columns** |

### Returning Different Types

**Scalar:**
```sql
CREATE FUNCTION posts_word_count(p posts) RETURNS int AS $$
  SELECT array_length(regexp_split_to_array(p.body, '\s+'), 1)
$$ LANGUAGE sql STABLE;
```

**Another row type:**
```sql
CREATE FUNCTION posts_author(p posts) RETURNS users AS $$
  SELECT * FROM users WHERE id = p.author_id
$$ LANGUAGE sql STABLE;
```

**Multiple rows (connection):**
```sql
CREATE FUNCTION users_recent_posts(u users) RETURNS SETOF posts AS $$
  SELECT * FROM posts
  WHERE author_id = u.id
  ORDER BY created_at DESC
  LIMIT 10
$$ LANGUAGE sql STABLE;
```

### Performance Considerations

**Inline-able functions** (best performance):
```sql
-- SQL language, simple expression - can be inlined
CREATE FUNCTION users_full_name(u users) RETURNS text AS $$
  SELECT u.first_name || ' ' || u.last_name
$$ LANGUAGE sql STABLE;
```

**Non-inline functions** (function call overhead):
```sql
-- PL/pgSQL can't be inlined
CREATE FUNCTION users_expensive_calc(u users) RETURNS int AS $$
BEGIN
  -- Complex logic
  RETURN result;
END;
$$ LANGUAGE plpgsql STABLE;
```

If a computed column is expensive, consider caching with a materialized view or trigger-updated column.

## Custom Query Functions

Functions become root Query fields when they:
1. Are `STABLE` or `IMMUTABLE`
2. Don't match the computed column pattern

### Single Row Return

```sql
CREATE FUNCTION current_user_profile() RETURNS users AS $$
  SELECT * FROM users WHERE id = current_user_id()
$$ LANGUAGE sql STABLE;
```

```graphql
type Query {
  currentUserProfile: User
}
```

### Multiple Rows (Connection)

```sql
CREATE FUNCTION search_users(query text) RETURNS SETOF users AS $$
  SELECT * FROM users
  WHERE name ILIKE '%' || query || '%'
     OR email ILIKE '%' || query || '%'
$$ LANGUAGE sql STABLE;
```

```graphql
type Query {
  searchUsers(
    query: String!
    first: Int
    after: Cursor
    # ... pagination args
  ): UsersConnection
}
```

### Named Arguments (Important!)

Always use named arguments. Unnamed arguments produce poor GraphQL:

```sql
-- BAD: unnamed arguments
CREATE FUNCTION search(text, int) RETURNS SETOF posts AS $$ ... $$;
-- Produces: search(arg0: String!, arg1: Int!)

-- GOOD: named arguments
CREATE FUNCTION search(query text, limit_count int DEFAULT 10) RETURNS SETOF posts AS $$ ... $$;
-- Produces: search(query: String!, limitCount: Int)
```

### Default Argument Values

```sql
CREATE FUNCTION get_posts(
  status post_status DEFAULT 'published',
  limit_count int DEFAULT 20
) RETURNS SETOF posts AS $$
  SELECT * FROM posts
  WHERE status = get_posts.status
  LIMIT limit_count
$$ LANGUAGE sql STABLE;
```

```graphql
type Query {
  getPosts(status: PostStatus, limitCount: Int): PostsConnection
}
```

## Custom Mutation Functions

Functions become Mutation fields when `VOLATILE`.

### Basic Mutation

```sql
CREATE FUNCTION register_user(
  email text,
  password text,
  name text
) RETURNS users AS $$
DECLARE
  new_user users;
BEGIN
  INSERT INTO users (email, password_hash, name)
  VALUES (email, crypt(password, gen_salt('bf')), name)
  RETURNING * INTO new_user;

  RETURN new_user;
END;
$$ LANGUAGE plpgsql VOLATILE;
```

```graphql
type Mutation {
  registerUser(input: RegisterUserInput!): RegisterUserPayload
}

input RegisterUserInput {
  email: String!
  password: String!
  name: String!
}

type RegisterUserPayload {
  user: User
  query: Query  # Access to root query
}
```

### Relay Edge Fields: A Key Limitation

**Problem:** Custom PostgreSQL functions returning a row type only produce a payload with the node—no edge field. This breaks Relay's `@appendEdge` directive.

Auto-generated CRUD mutations include edge fields:
```graphql
type CreateMessagePayload {
  message: Message
  messageEdge(orderBy: MessagesOrderBy): MessagesEdge  # Relay can use this
  query: Query
}
```

Custom function mutations do NOT:
```graphql
type SendMessagePayload {
  message: Message  # No edge field - @appendEdge won't work
  query: Query
}
```

**Solutions:**

1. **Use auto-generated mutations when possible** — They include edge fields automatically.

2. **Return the junction/connection table row** — If adding to a connection (e.g., `message_channels`), return that row instead:
   ```sql
   -- Instead of returning messages, return the junction row
   CREATE FUNCTION send_message_to_channel(channel_id int, content text)
   RETURNS message_channels AS $$  -- Returns junction, not message
     ...
   $$ LANGUAGE plpgsql VOLATILE;
   ```

3. **Use `makeExtendSchemaPlugin` for proper payloads** — Build the edge field yourself in JavaScript:
   ```js
   // plugins/sendMessage.js
   import { makeExtendSchemaPlugin, gql } from "postgraphile/utils";

   export const SendMessagePlugin = makeExtendSchemaPlugin({
     typeDefs: gql`
       extend type Mutation {
         sendMessage(channelId: ID!, content: String!): SendMessagePayload
       }
       type SendMessagePayload {
         message: Message
         messageEdge: MessagesEdge  # Now Relay can use @appendEdge
         query: Query
       }
     `,
     // ... plan resolvers to build the edge
   });
   ```

4. **Accept manual cache updates** — Use Relay's imperative `updater` function instead of declarative `@appendEdge`.

**Best practice:** If your mutation adds items to a list that clients need to display immediately, prefer auto-generated mutations or build proper edge fields with `makeExtendSchemaPlugin`.

### Returning Multiple Values

Use `OUT` parameters or a custom type:

```sql
-- Using OUT parameters
CREATE FUNCTION login(
  email text,
  password text,
  OUT user_record users,
  OUT token text
) AS $$
BEGIN
  SELECT * INTO user_record FROM users WHERE users.email = login.email;
  IF user_record IS NULL OR NOT verify_password(password, user_record.password_hash) THEN
    RAISE EXCEPTION 'Invalid credentials';
  END IF;
  token := generate_jwt(user_record.id);
END;
$$ LANGUAGE plpgsql VOLATILE;
```

### Void Returns

For side-effect-only mutations:

```sql
CREATE FUNCTION send_password_reset(email text) RETURNS void AS $$
BEGIN
  -- Queue email, etc.
  PERFORM pg_notify('email', json_build_object('type', 'password_reset', 'email', email)::text);
END;
$$ LANGUAGE plpgsql VOLATILE;
```

```graphql
type Mutation {
  sendPasswordReset(input: SendPasswordResetInput!): SendPasswordResetPayload
}

type SendPasswordResetPayload {
  query: Query  # Only query field, no return value
}
```

### Transaction Handling

Mutations run in a transaction. For explicit control:

```sql
CREATE FUNCTION transfer_funds(
  from_account_id int,
  to_account_id int,
  amount numeric
) RETURNS void AS $$
BEGIN
  -- Both operations in same transaction (automatic)
  UPDATE accounts SET balance = balance - amount WHERE id = from_account_id;
  UPDATE accounts SET balance = balance + amount WHERE id = to_account_id;

  -- Validate
  IF (SELECT balance FROM accounts WHERE id = from_account_id) < 0 THEN
    RAISE EXCEPTION 'Insufficient funds';
  END IF;
END;
$$ LANGUAGE plpgsql VOLATILE;
```

## SECURITY DEFINER vs INVOKER

### SECURITY INVOKER (Default)

Function runs with permissions of the calling role:

```sql
CREATE FUNCTION my_func() RETURNS void AS $$ ... $$
LANGUAGE plpgsql VOLATILE SECURITY INVOKER;
```

RLS policies apply normally.

### SECURITY DEFINER

Function runs with permissions of the function owner:

```sql
CREATE FUNCTION admin_only_action() RETURNS void AS $$
BEGIN
  -- Runs as function owner, bypasses RLS
  UPDATE sensitive_table SET ...;
END;
$$ LANGUAGE plpgsql VOLATILE SECURITY DEFINER;
```

**Caution:** `SECURITY DEFINER` bypasses RLS. Use sparingly and validate inputs carefully.

Best practice - revoke public execute:
```sql
REVOKE ALL ON FUNCTION admin_only_action FROM PUBLIC;
GRANT EXECUTE ON FUNCTION admin_only_action TO admin_role;
```

## Plugin-Based Extensions

For complex logic that doesn't fit in SQL functions, use `makeExtendSchemaPlugin`.

### Basic Extension

```js
import { makeExtendSchemaPlugin, gql } from "postgraphile/utils";

export const MyExtensionPlugin = makeExtendSchemaPlugin({
  typeDefs: gql`
    extend type Query {
      serverTime: String!
    }
    extend type Mutation {
      complexAction(input: ComplexInput!): ComplexPayload
    }
    input ComplexInput {
      data: JSON!
    }
    type ComplexPayload {
      success: Boolean!
      message: String
    }
  `,
  resolvers: {
    Query: {
      serverTime() {
        return new Date().toISOString();
      },
    },
    Mutation: {
      async complexAction(_, { input }, context) {
        // Access pgClient for database operations
        const { pgClient } = context;
        await pgClient.query('BEGIN');
        try {
          // ... complex logic
          await pgClient.query('COMMIT');
          return { success: true };
        } catch (e) {
          await pgClient.query('ROLLBACK');
          return { success: false, message: e.message };
        }
      },
    },
  },
});
```

### Using @pgQuery for Database Integration

The `@pgQuery` directive integrates with PostGraphile's look-ahead system:

```js
export const SearchPlugin = makeExtendSchemaPlugin({
  typeDefs: gql`
    extend type Query {
      advancedSearch(query: String!): [User!]! @pgQuery(
        source: "users"
        withQueryBuilder: true
      )
    }
  `,
  plans: {
    Query: {
      advancedSearch($parent, args, info) {
        // Use Grafast for efficient query building
        return connection(/* ... */);
      },
    },
  },
});
```

## Function Control Tags

### Hide Function

```sql
-- V5 (recommended)
COMMENT ON FUNCTION internal_helper() IS '@behavior -queryField -mutationField';

-- V4 preset only
COMMENT ON FUNCTION internal_helper() IS '@omit';
```

### Rename Function

```sql
COMMENT ON FUNCTION search_users IS '@name findPeople';
```

### Control Behavior (V5)

```sql
-- Only as query, not mutation even if volatile
COMMENT ON FUNCTION my_func IS '@behavior +queryField -mutationField';

-- Only as computed column, not root field
COMMENT ON FUNCTION users_stats IS '@behavior +typeField -queryField';
```

## Common Patterns

### Upsert

```sql
CREATE FUNCTION upsert_user_preference(
  user_id int,
  key text,
  value jsonb
) RETURNS user_preferences AS $$
  INSERT INTO user_preferences (user_id, key, value)
  VALUES (user_id, key, value)
  ON CONFLICT (user_id, key)
  DO UPDATE SET value = EXCLUDED.value, updated_at = now()
  RETURNING *
$$ LANGUAGE sql VOLATILE;
```

### Batch Operations

```sql
CREATE FUNCTION bulk_update_status(
  ids int[],
  new_status post_status
) RETURNS SETOF posts AS $$
  UPDATE posts
  SET status = new_status, updated_at = now()
  WHERE id = ANY(ids)
  RETURNING *
$$ LANGUAGE sql VOLATILE;
```

### Computed with Parameters

```sql
-- Computed column with extra parameter
CREATE FUNCTION posts_truncated_body(p posts, max_length int DEFAULT 100)
RETURNS text AS $$
  SELECT CASE
    WHEN length(p.body) > max_length
    THEN left(p.body, max_length) || '...'
    ELSE p.body
  END
$$ LANGUAGE sql STABLE;
```

```graphql
type Post {
  truncatedBody(maxLength: Int): String
}
```

## Next Steps

- [security.md](security.md) — RLS integration with functions
- [smart-tags.md](smart-tags.md) — Control function exposure
