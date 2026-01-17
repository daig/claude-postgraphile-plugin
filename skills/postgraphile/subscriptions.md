# Subscriptions: LISTEN/NOTIFY Realtime

PostGraphile V5 has built-in support for GraphQL subscriptions using PostgreSQL's `LISTEN/NOTIFY` mechanism. This enables real-time updates without external message queues.

## How It Works

```
┌──────────┐  NOTIFY  ┌────────────┐  WebSocket  ┌──────────┐
│PostgreSQL│─────────▶│PostGraphile│────────────▶│  Client  │
└──────────┘          └────────────┘             └──────────┘
```

1. PostgreSQL trigger calls `pg_notify()` on data change
2. PostGraphile's built-in `pgSubscriber` listens on specified channels
3. PostGraphile pushes updates to subscribed clients via WebSocket

## Setup

### V5 Built-in Subscriptions

V5 includes `pgSubscriber` in the Grafast context by default when you enable `pubsub` on your pgService:

```js
// graphile.config.mjs
import { PostGraphileAmberPreset } from "postgraphile/presets/amber";
import { makePgService } from "postgraphile/adaptors/pg";

export default {
  extends: [PostGraphileAmberPreset],
  pgServices: [makePgService({
    connectionString: process.env.DATABASE_URL,
    pubsub: true,  // Enables built-in pgSubscriber
  })],
  grafserv: {
    websockets: true,  // Enable WebSocket support
  },
};
```

### Optional: pg-pubsub Plugin (Advanced)

For advanced features like Redis pubsub adapter or custom subscriber implementations:

```bash
npm install @graphile/pg-pubsub
```

```js
import { PgPubsubPlugin } from "@graphile/pg-pubsub";

export default {
  plugins: [PgPubsubPlugin],
  // ...
};
```

## Simple Subscriptions

> **Note:** V5 does not include "simple subscriptions" by default. The `simpleSubscriptions` option from V4 was removed. For quick subscriptions, use the custom subscription pattern below with `listen()`.

If you need V4-style simple subscriptions, a community plugin may be available, but the recommended approach is to define explicit subscription types using `makeExtendSchemaPlugin`.

## Custom Subscriptions

For production, define custom subscription types using Grafast's `listen()` step.

### Step 1: Create Notify Trigger

```sql
-- Function to notify on post changes
CREATE FUNCTION notify_post_change() RETURNS trigger AS $$
DECLARE
  payload json;
BEGIN
  payload := json_build_object(
    'event', TG_OP,
    'id', COALESCE(NEW.id, OLD.id),
    'author_id', COALESCE(NEW.author_id, OLD.author_id)
  );

  -- Notify on the channel
  PERFORM pg_notify(
    'postgraphile:post_change',
    payload::text
  );

  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

-- Attach trigger
CREATE TRIGGER post_change_trigger
  AFTER INSERT OR UPDATE OR DELETE ON posts
  FOR EACH ROW
  EXECUTE FUNCTION notify_post_change();
```

### Step 2: Define Subscription Plugin (V5 Pattern)

V5 uses Grafast's `listen()` step with `subscribePlan()` returning a streamable step and `plan()` transforming each event:

```js
// plugins/subscriptions.js
import { makeExtendSchemaPlugin, gql } from "postgraphile/utils";
import { context, listen, lambda } from "postgraphile/grafast";
import { jsonParse } from "postgraphile/@dataplan/json";

export const PostSubscriptionsPlugin = makeExtendSchemaPlugin({
  typeDefs: gql`
    type PostChangePayload {
      event: String!
      postId: Int!
    }

    extend type Subscription {
      postChanged: PostChangePayload
    }
  `,
  plans: {
    Subscription: {
      postChanged: {
        // subscribePlan returns a streamable step (the event source)
        subscribePlan() {
          const $pgSubscriber = context().get("pgSubscriber");
          // listen() returns a stream of raw payloads
          return listen($pgSubscriber, "postgraphile:post_change", jsonParse);
        },
        // plan() transforms each event payload into the GraphQL response
        plan($event) {
          return {
            event: $event.get("event"),
            postId: $event.get("id"),
          };
        },
      },
    },
  },
});
```

### Dynamic Topics with lambda()

For subscriptions to dynamic channels (e.g., per-entity):

```js
import { context, listen, lambda } from "postgraphile/grafast";
import { jsonParse } from "postgraphile/@dataplan/json";

subscribePlan($parent, { $entityId }) {
  const $pgSubscriber = context().get("pgSubscriber");
  // Build topic name from argument
  const $topic = lambda($entityId, (id) => `entity:${id}`);
  return listen($pgSubscriber, $topic, jsonParse);
}
```

### Step 3: Use in Config

```js
// graphile.config.mjs
import { PostSubscriptionsPlugin } from "./plugins/subscriptions.js";

export default {
  plugins: [PostSubscriptionsPlugin],
  pgServices: [makePgService({
    connectionString: process.env.DATABASE_URL,
    pubsub: true,
  })],
  grafserv: {
    websockets: true,
  },
};
```

## Filtering Subscriptions

Allow clients to subscribe to specific events using Grafast's `filter()` step:

```js
import { context, listen, filter, lambda } from "postgraphile/grafast";
import { jsonParse } from "postgraphile/@dataplan/json";

export const FilteredSubscriptionPlugin = makeExtendSchemaPlugin({
  typeDefs: gql`
    extend type Subscription {
      postChangedByAuthor(authorId: Int!): PostChangePayload
    }
  `,
  plans: {
    Subscription: {
      postChangedByAuthor: {
        subscribePlan($parent, { $authorId }) {
          const $pgSubscriber = context().get("pgSubscriber");
          const $events = listen($pgSubscriber, "postgraphile:post_change", jsonParse);

          // Filter events by author_id using Grafast filter step
          return filter($events, ($event) => {
            const $eventAuthorId = $event.get("author_id");
            return lambda([$eventAuthorId, $authorId], ([eventAuthor, targetAuthor]) =>
              eventAuthor === targetAuthor
            );
          });
        },
        plan($event) {
          return {
            event: $event.get("event"),
            postId: $event.get("id"),
          };
        },
      },
    },
  },
});
```

### Alternative: Filter at Database Level

For better performance, use specific channels per author:

```sql
-- Notify on author-specific channel
PERFORM pg_notify('post_change:author:' || NEW.author_id, payload::text);
```

```js
subscribePlan($parent, { $authorId }) {
  const $pgSubscriber = context().get("pgSubscriber");
  const $topic = lambda($authorId, (id) => `post_change:author:${id}`);
  return listen($pgSubscriber, $topic, jsonParse);
}
```

## Payload Size Limit

PostgreSQL `NOTIFY` payloads are limited to ~8000 bytes. For large data:

1. **Send IDs only**, let client re-fetch:
   ```sql
   PERFORM pg_notify('post_change', json_build_object('id', NEW.id)::text);
   ```

2. **Use message table** for large payloads:
   ```sql
   INSERT INTO notification_queue (channel, payload) VALUES ('post_change', large_payload);
   PERFORM pg_notify('post_change', json_build_object('queue_id', NEW.id)::text);
   ```

## Multiple Channels

Subscribe to multiple channels:

```js
subscribePlan() {
  const $pgSubscriber = context().get("pgSubscriber");
  return subscribe($pgSubscriber, [
    "postgraphile:post_created",
    "postgraphile:post_updated",
    "postgraphile:post_deleted",
  ]);
}
```

## Connection Management

### Heartbeats

WebSocket connections need keepalive:

```js
// graphile.config.mjs
export default {
  grafserv: {
    websockets: true,
    websocketKeepAlive: 30000,  // 30 second heartbeat
  },
};
```

### Handling Disconnections

PostgreSQL LISTEN connections can drop. The built-in pubsub handles reconnection automatically.

For custom handling with the `@graphile/pg-pubsub` plugin:

```js
pgServices: [makePgService({
  connectionString: process.env.DATABASE_URL,
  pubsub: {
    retryOnInitFail: true,
    reconnectDelay: 1000,  // ms
  },
})],
```

## Security

### Channel Namespacing

Use prefixed channels to avoid conflicts:

```sql
-- Use app-specific prefix
PERFORM pg_notify('myapp:user:' || user_id::text, payload);
```

### Authorization

Check permissions in subscribe plan:

```js
subscribePlan($parent, { $userId }) {
  const $currentUserId = context().get("currentUserId");

  // Only subscribe to own events
  if ($userId !== $currentUserId) {
    throw new Error("Cannot subscribe to other users");
  }

  return subscribe(/* ... */);
}
```

### RLS and Subscriptions

Subscriptions bypass RLS by default (they're push-based). Enforce access in the plan:

```js
plan($event) {
  const $postId = $event.get("id");
  // Fetch through normal GraphQL which applies RLS
  return loadOne($postId, /* ... */);
}
```

## Low-Level Plugin Approach

For direct schema hook access instead of `makeExtendSchemaPlugin`:

```js
import { listen, context } from "postgraphile/grafast";
import { jsonParse } from "postgraphile/@dataplan/json";

export const SubscriptionPlugin = {
  name: "SubscriptionPlugin",
  schema: {
    hooks: {
      build(build) {
        // Register subscription using build hooks
        return build;
      },
      GraphQLObjectType_fields(fields, build, context) {
        if (context.Self.name !== "Subscription") return fields;

        return build.extend(fields, {
          postUpdated: {
            type: build.getTypeByName("Post"),
            subscribePlan() {
              const $subscriber = context().get("pgSubscriber");
              return listen($subscriber, "post_updated", jsonParse);
            },
            plan($event) {
              return $event.get("post");
            },
          },
        });
      },
    },
  },
};
```

## Alternative: External Pubsub

For high-scale or multi-server deployments, use external pubsub:

```js
import { createClient } from "redis";

const redis = createClient({ url: process.env.REDIS_URL });

export const RedisSubscriptionPlugin = makeExtendSchemaPlugin({
  // Use Redis pub/sub instead of pg_notify
  plans: {
    Subscription: {
      myEvent: {
        subscribePlan() {
          return listenToRedis(redis, "my-channel");
        },
      },
    },
  },
});
```

## Common Patterns

### Presence (User Online Status)

```sql
-- Trigger on user last_seen update
CREATE FUNCTION notify_user_presence() RETURNS trigger AS $$
BEGIN
  PERFORM pg_notify(
    'user_presence',
    json_build_object('user_id', NEW.id, 'online', NEW.last_seen > now() - interval '5 minutes')::text
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### Chat Messages

```sql
CREATE FUNCTION notify_new_message() RETURNS trigger AS $$
BEGIN
  -- Notify the conversation channel
  PERFORM pg_notify(
    'conversation:' || NEW.conversation_id,
    json_build_object('message_id', NEW.id, 'sender_id', NEW.sender_id)::text
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

```js
// Subscribe to specific conversation
import { context, listen, lambda } from "postgraphile/grafast";
import { jsonParse } from "postgraphile/@dataplan/json";

subscribePlan($parent, { $conversationId }) {
  const $pgSubscriber = context().get("pgSubscriber");
  const $topic = lambda($conversationId, (id) => `conversation:${id}`);
  return listen($pgSubscriber, $topic, jsonParse);
}
```

### Notifications Feed

```sql
CREATE FUNCTION notify_user_notification() RETURNS trigger AS $$
BEGIN
  PERFORM pg_notify(
    'user_notifications:' || NEW.user_id,
    json_build_object('notification_id', NEW.id, 'type', NEW.type)::text
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## Debugging

### Test NOTIFY manually

```sql
-- In one psql session:
LISTEN postgraphile:test;

-- In another:
SELECT pg_notify('postgraphile:test', '{"test": true}');

-- First session shows:
-- Asynchronous notification "postgraphile:test" with payload "{"test": true}" received
```

### Check Active Subscriptions

In PostGraphile logs (with debug enabled):

```js
export default {
  grafserv: {
    websockets: true,
  },
  // Enable debug logging
};
```

## Next Steps

- [plugins.md](plugins.md) — Configure pg-pubsub and WebSockets
- [security.md](security.md) — Secure subscription channels
