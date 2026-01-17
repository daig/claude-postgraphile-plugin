# PostGraphile V5 Skill for Claude Code

A Claude Code plugin providing patterns for generating GraphQL APIs from PostgreSQL using PostGraphile V5 (Release Candidate).

## Installation

```bash
# From marketplace
claude plugin add daig/claude-postgraphile-plugin

# Or add to settings
{
  "plugins": ["github:daig/claude-postgraphile-plugin"]
}
```

## What's Covered

### Schema Translation
- PostgreSQL tables, columns, types → GraphQL types and fields
- Foreign keys → relationship fields
- Enums, composite types, arrays
- Naming conventions and inflection

### Smart Tags & Behaviors
- `@name`, `@fieldName`, `@foreignFieldName` for renaming
- `@behavior` system for fine-grained control (V5)
- `@deprecated` for deprecation notices
- `@primaryKey`, `@unique`, `@foreignKey` for views

### Polymorphism (V5)
- `@interface mode:single` - single table inheritance
- `@interface mode:relational` - joined table inheritance
- `@interface mode:union` - independent tables via composite type
- `@implements`, `@unionMember` for type membership

### Functions
- Computed columns via `tablename_fieldname(row)` pattern
- Custom queries (`STABLE` functions)
- Custom mutations (`VOLATILE` functions)
- `SECURITY DEFINER` vs `SECURITY INVOKER`

### Security
- Row-Level Security (RLS) integration
- `pgSettings` for passing JWT claims
- `current_user_id()` helper pattern
- Role-based and tenant-based policies

### Subscriptions
- Built-in `pgSubscriber` (V5)
- Grafast `listen()` step pattern
- Dynamic topics with `lambda()`
- LISTEN/NOTIFY triggers

### Configuration
- Installation: `npm install postgraphile@rc`
- Preset system (Amber, V4 compatibility, Relay)
- Grafast engine and optimizations
- Community plugins

## Prerequisites

This skill assumes knowledge of:
- SQL fundamentals (queries, joins, transactions)
- PostgreSQL (RLS, functions, types)
- GraphQL basics (types, queries, mutations)

## File Structure

```
skills/postgraphile/
├── SKILL.md          # Quick reference and decision tables
├── schema.md         # PostgreSQL → GraphQL translation
├── smart-tags.md     # @behavior, @name, polymorphism tags
├── functions.md      # Computed columns, custom queries/mutations
├── security.md       # RLS + pgSettings + JWT patterns
├── subscriptions.md  # LISTEN/NOTIFY realtime
└── plugins.md        # Configuration, presets, installation
```

## Quick Reference

### How do I customize naming?
- `@name NewName` - Rename any entity
- `@fieldName author` - Rename FK relation field
- `pg-simplify-inflector` - Cleaner names globally

### How do I hide something?
- `@behavior -*` - Hide completely (V5)
- `@behavior -insert -update -delete` - Hide mutations only

### Which function type?
- `STABLE` → Query field (reads data)
- `VOLATILE` → Mutation field (writes data)
- `tablename_fieldname(row)` → Computed column

### Security pattern?
```sql
CREATE FUNCTION current_user_id() RETURNS int AS $$
  SELECT nullif(current_setting('myapp.user_id', true), '')::int
$$ LANGUAGE sql STABLE;

CREATE POLICY posts_owner ON posts
  USING (author_id = current_user_id());
```

## License

MIT
