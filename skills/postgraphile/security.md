# Security: RLS, pgSettings, and JWT Integration

PostGraphile's security model uses PostgreSQL's Row-Level Security (RLS) with request context passed via `pgSettings`. This keeps authorization logic in the database, ensuring consistency across all access paths.

## Architecture Overview

```
┌──────────┐      ┌──────────────┐      ┌────────────┐
│  Client  │─────▶│ PostGraphile │─────▶│ PostgreSQL │
│  (JWT)   │      │  (Proxy)     │      │   (RLS)    │
└──────────┘      └──────────────┘      └────────────┘
                         │
                         │ SET LOCAL myapp.user_id = '123'
                         │ SET LOCAL myapp.role = 'user'
                         ▼
                  ┌──────────────┐
                  │  pgSettings  │
                  └──────────────┘
```

1. Client sends JWT with request
2. PostGraphile validates JWT, extracts claims
3. PostGraphile sets `pgSettings` (session variables) before each query
4. PostgreSQL RLS policies use these settings to filter data

## pgSettings Configuration

### Basic Setup

```js
// graphile.config.mjs
import { PostGraphileAmberPreset } from "postgraphile/presets/amber";
import { makePgService } from "postgraphile/adaptors/pg";

export default {
  extends: [PostGraphileAmberPreset],
  pgServices: [makePgService({
    connectionString: process.env.DATABASE_URL,
    pgSettings: async (ctx) => {
      // ctx contains request info (from your server adapter)
      const user = ctx.user; // Set by your auth middleware
      return {
        'myapp.user_id': user?.id?.toString() ?? '',
        'myapp.role': user?.role ?? 'anonymous',
        'myapp.tenant_id': user?.tenantId?.toString() ?? '',
      };
    },
  })],
};
```

### What pgSettings Does

For each request, PostGraphile executes:

```sql
BEGIN;
SET LOCAL myapp.user_id TO '123';
SET LOCAL myapp.role TO 'user';
SET LOCAL myapp.tenant_id TO '456';
-- Your GraphQL query runs here
COMMIT;
```

`SET LOCAL` ensures settings only apply to the current transaction.

## Helper Functions

Create functions to access settings cleanly:

```sql
-- Current user ID (returns NULL for anonymous)
CREATE FUNCTION current_user_id() RETURNS int AS $$
  SELECT nullif(current_setting('myapp.user_id', true), '')::int
$$ LANGUAGE sql STABLE;

-- Current user role
CREATE FUNCTION current_user_role() RETURNS text AS $$
  SELECT coalesce(nullif(current_setting('myapp.role', true), ''), 'anonymous')
$$ LANGUAGE sql STABLE;

-- Current tenant ID
CREATE FUNCTION current_tenant_id() RETURNS int AS $$
  SELECT nullif(current_setting('myapp.tenant_id', true), '')::int
$$ LANGUAGE sql STABLE;

-- Is current user an admin?
CREATE FUNCTION is_admin() RETURNS boolean AS $$
  SELECT current_user_role() = 'admin'
$$ LANGUAGE sql STABLE;
```

The second argument `true` in `current_setting(..., true)` returns NULL instead of error if setting doesn't exist.

## RLS Policy Patterns

### User Owns Resource

```sql
-- Users can only see their own posts
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

CREATE POLICY posts_select ON posts
  FOR SELECT
  USING (author_id = current_user_id());

CREATE POLICY posts_insert ON posts
  FOR INSERT
  WITH CHECK (author_id = current_user_id());

CREATE POLICY posts_update ON posts
  FOR UPDATE
  USING (author_id = current_user_id())
  WITH CHECK (author_id = current_user_id());

CREATE POLICY posts_delete ON posts
  FOR DELETE
  USING (author_id = current_user_id());
```

### Multi-Tenant Isolation

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY orders_tenant_isolation ON orders
  USING (tenant_id = current_tenant_id());
```

### Role-Based Access

```sql
ALTER TABLE admin_settings ENABLE ROW LEVEL SECURITY;

-- Only admins can see
CREATE POLICY admin_settings_select ON admin_settings
  FOR SELECT
  USING (is_admin());

-- Only admins can modify
CREATE POLICY admin_settings_all ON admin_settings
  FOR ALL
  USING (is_admin())
  WITH CHECK (is_admin());
```

### Public Read, Authenticated Write

```sql
ALTER TABLE articles ENABLE ROW LEVEL SECURITY;

-- Anyone can read published articles
CREATE POLICY articles_public_read ON articles
  FOR SELECT
  USING (status = 'published' OR author_id = current_user_id());

-- Only author can insert/update/delete
CREATE POLICY articles_author_write ON articles
  FOR ALL
  USING (author_id = current_user_id())
  WITH CHECK (author_id = current_user_id());
```

### Hierarchical Access (Manager sees team)

```sql
-- Employees table with manager hierarchy
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;

CREATE POLICY employees_access ON employees
  FOR SELECT
  USING (
    id = current_user_id()  -- See yourself
    OR manager_id = current_user_id()  -- See direct reports
    OR is_admin()  -- Admins see all
  );
```

## JWT Integration

### Using lazy-jwt Preset (Simple)

For simple JWT validation:

```js
// graphile.config.mjs
import { PostGraphileAmberPreset } from "postgraphile/presets/amber";
import { PostGraphileLazyJWTPreset } from "postgraphile/presets/lazy-jwt";

export default {
  extends: [PostGraphileAmberPreset, PostGraphileLazyJWTPreset],
  jwt: {
    secret: process.env.JWT_SECRET,
    // Map JWT claims to pgSettings
    pgSettings: (claims) => ({
      'myapp.user_id': claims.sub,
      'myapp.role': claims.role,
    }),
  },
};
```

### Custom JWT Middleware (Production)

For more control, handle JWT in your server:

```js
// server.js
import express from "express";
import jwt from "jsonwebtoken";
import { postgraphile } from "postgraphile";
import preset from "./graphile.config.mjs";

const app = express();

// JWT middleware
app.use((req, res, next) => {
  const auth = req.headers.authorization;
  if (auth?.startsWith('Bearer ')) {
    try {
      const token = auth.slice(7);
      req.user = jwt.verify(token, process.env.JWT_SECRET);
    } catch (e) {
      // Invalid token - continue as anonymous
    }
  }
  next();
});

// PostGraphile with context
app.use(postgraphile(preset, {
  // User is available in pgSettings via ctx
}));
```

### Generating JWTs from PostgreSQL

PostGraphile can return JWTs from mutations:

```sql
-- Create a type for JWT payload
CREATE TYPE jwt_token AS (
  sub text,
  role text,
  exp bigint
);

-- Login function returns JWT
CREATE FUNCTION login(email text, password text) RETURNS jwt_token AS $$
DECLARE
  user_record users;
BEGIN
  SELECT * INTO user_record FROM users WHERE users.email = login.email;

  IF user_record IS NULL THEN
    RAISE EXCEPTION 'Invalid email or password';
  END IF;

  IF NOT user_record.password_hash = crypt(password, user_record.password_hash) THEN
    RAISE EXCEPTION 'Invalid email or password';
  END IF;

  RETURN (
    user_record.id::text,
    user_record.role,
    extract(epoch from now() + interval '7 days')::bigint
  )::jwt_token;
END;
$$ LANGUAGE plpgsql VOLATILE SECURITY DEFINER;
```

Configure PostGraphile to sign the JWT:

```js
// graphile.config.mjs
export default {
  // ...
  jwt: {
    secret: process.env.JWT_SECRET,
    signOptions: {
      audience: 'postgraphile',
      issuer: 'myapp',
    },
  },
  schema: {
    jwtType: ['public', 'jwt_token'],  // [schema, type_name]
  },
};
```

## Connection Pooling Considerations

### With RLS and pgSettings

When using connection pooling (PgBouncer), ensure `SET LOCAL` works:

1. Use transaction mode pooling (not statement mode)
2. Ensure connections are properly reset between transactions

```ini
# pgbouncer.ini
pool_mode = transaction
```

### PostGraphile's Built-in Pool

PostGraphile manages connections efficiently by default. For high traffic:

```js
// graphile.config.mjs
export default {
  pgServices: [makePgService({
    connectionString: process.env.DATABASE_URL,
    poolConfig: {
      max: 20,  // Maximum connections
      idleTimeoutMillis: 30000,
    },
  })],
};
```

## Security Best Practices

### 1. Always Enable RLS

```sql
-- Enable RLS on all tables with sensitive data
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;
-- etc.

-- Force RLS for table owner too (optional, for testing)
ALTER TABLE users FORCE ROW LEVEL SECURITY;
```

### 2. Default Deny

Create restrictive default, then allow:

```sql
-- No policies = deny all (when RLS enabled)
ALTER TABLE secrets ENABLE ROW LEVEL SECURITY;

-- Explicitly grant what's needed
CREATE POLICY secrets_admin_only ON secrets
  USING (is_admin());
```

### 3. Hide Sensitive Columns

```sql
-- V5 (recommended)
COMMENT ON COLUMN users.password_hash IS '@behavior -select';
COMMENT ON COLUMN users.reset_token IS '@behavior -select';

-- V4 preset only
COMMENT ON COLUMN users.password_hash IS '@omit';
COMMENT ON COLUMN users.reset_token IS '@omit';
```

### 4. Validate in Mutations

```sql
CREATE FUNCTION create_post(title text, body text) RETURNS posts AS $$
DECLARE
  new_post posts;
BEGIN
  -- Explicit validation
  IF current_user_id() IS NULL THEN
    RAISE EXCEPTION 'Must be logged in';
  END IF;

  IF length(title) < 3 THEN
    RAISE EXCEPTION 'Title too short';
  END IF;

  INSERT INTO posts (author_id, title, body)
  VALUES (current_user_id(), title, body)
  RETURNING * INTO new_post;

  RETURN new_post;
END;
$$ LANGUAGE plpgsql VOLATILE;
```

### 5. Audit Logging

```sql
CREATE TABLE audit_log (
  id serial PRIMARY KEY,
  table_name text NOT NULL,
  action text NOT NULL,
  row_id int,
  user_id int,
  old_data jsonb,
  new_data jsonb,
  timestamp timestamptz DEFAULT now()
);

CREATE FUNCTION audit_trigger() RETURNS trigger AS $$
BEGIN
  INSERT INTO audit_log (table_name, action, row_id, user_id, old_data, new_data)
  VALUES (
    TG_TABLE_NAME,
    TG_OP,
    COALESCE(NEW.id, OLD.id),
    current_user_id(),
    CASE WHEN TG_OP IN ('UPDATE', 'DELETE') THEN to_jsonb(OLD) END,
    CASE WHEN TG_OP IN ('INSERT', 'UPDATE') THEN to_jsonb(NEW) END
  );
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER posts_audit
  AFTER INSERT OR UPDATE OR DELETE ON posts
  FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

### 6. Use Separate Database Roles

```sql
-- Role for PostGraphile connections (limited)
CREATE ROLE postgraphile_app LOGIN PASSWORD 'secret';
GRANT CONNECT ON DATABASE myapp TO postgraphile_app;
GRANT USAGE ON SCHEMA public TO postgraphile_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO postgraphile_app;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO postgraphile_app;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA public TO postgraphile_app;

-- Don't grant superuser or table ownership
```

## Debugging RLS

### Check Current Settings

```sql
SELECT current_setting('myapp.user_id', true) AS user_id,
       current_setting('myapp.role', true) AS role,
       current_setting('myapp.tenant_id', true) AS tenant_id;
```

### Check Policy Definitions

```sql
SELECT tablename, policyname, permissive, roles, cmd, qual, with_check
FROM pg_policies
WHERE tablename = 'posts';
```

### Test as Specific User

```sql
BEGIN;
SET LOCAL myapp.user_id TO '42';
SELECT * FROM posts;  -- See what user 42 sees
ROLLBACK;
```

## Next Steps

- [functions.md](functions.md) — SECURITY DEFINER functions
- [smart-tags.md](smart-tags.md) — Hide columns with @behavior
