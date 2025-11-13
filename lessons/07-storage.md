# Lesson 7: State Management and Storage

**Duration**: 1.5 hours
**Level**: Intermediate

## Overview

Since Devvit functions are stateless, you need persistent storage to maintain data between executions. This lesson covers Redis storage, data patterns, and best practices.

## Why Redis?

Devvit provides **Redis** for persistent key-value storage:

- **Fast** - In-memory data store
- **Simple** - Key-value pairs
- **Persistent** - Data survives between function executions
- **Scalable** - Handled by Reddit's infrastructure

## Enabling Redis

Add to your configuration:

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
  redis: true,  // Enable Redis
});

export default Devvit;
```

## Basic Redis Operations

### Set and Get

**Store a value:**
```typescript
await context.redis.set('key', 'value');
await context.redis.set('username', 'alice');
await context.redis.set('count', '42');
```

**Retrieve a value:**
```typescript
const value = await context.redis.get('key');
// value is null if key doesn't exist
console.log(value); // 'value'

const username = await context.redis.get('username');
console.log(username); // 'alice'
```

**Important:** All values are stored as strings!

### Delete

**Delete a key:**
```typescript
await context.redis.del('key');
await context.redis.del('username');
```

**Delete multiple keys:**
```typescript
await context.redis.del(['key1', 'key2', 'key3']);
```

### Check Existence

**Check if key exists:**
```typescript
const exists = await context.redis.exists('key');
if (exists) {
  console.log('Key exists');
}
```

### Expiration

**Set with expiration (TTL):**
```typescript
// Expire in 3600 seconds (1 hour)
await context.redis.set('session', 'data', { expiration: new Date(Date.now() + 3600000) });
```

**Set expiration on existing key:**
```typescript
await context.redis.expire('key', new Date(Date.now() + 3600000));
```

## Numeric Operations

### Increment and Decrement

**Increment:**
```typescript
// Initialize counter
await context.redis.set('counter', '0');

// Increment by 1
const newValue = await context.redis.incrBy('counter', 1);
console.log(newValue); // 1

// Increment by 10
const value2 = await context.redis.incrBy('counter', 10);
console.log(value2); // 11
```

**Decrement:**
```typescript
const newValue = await context.redis.incrBy('counter', -1);
// Equivalent to decrementing by 1
```

**Atomic operations** - Perfect for counters and statistics!

## Storing Complex Data

### JSON Serialization

Redis stores strings, so use JSON for objects:

```typescript
interface UserData {
  username: string;
  score: number;
  lastSeen: number;
}

// Store object
const userData: UserData = {
  username: 'alice',
  score: 100,
  lastSeen: Date.now(),
};

await context.redis.set('user:alice', JSON.stringify(userData));

// Retrieve object
const dataStr = await context.redis.get('user:alice');
if (dataStr) {
  const data: UserData = JSON.parse(dataStr);
  console.log(data.username); // 'alice'
  console.log(data.score);    // 100
}
```

### Arrays

```typescript
// Store array
const items = ['apple', 'banana', 'cherry'];
await context.redis.set('fruits', JSON.stringify(items));

// Retrieve array
const fruitsStr = await context.redis.get('fruits');
const fruits: string[] = JSON.parse(fruitsStr || '[]');
console.log(fruits[0]); // 'apple'
```

### Maps/Dictionaries

```typescript
// Store map
const scores: { [key: string]: number } = {
  alice: 100,
  bob: 85,
  charlie: 92,
};

await context.redis.set('scores', JSON.stringify(scores));

// Retrieve map
const scoresStr = await context.redis.get('scores');
const scoresData = JSON.parse(scoresStr || '{}');
console.log(scoresData.alice); // 100
```

## Key Naming Conventions

Use descriptive, hierarchical key names:

```typescript
// Good naming patterns
await context.redis.set('user:alice:score', '100');
await context.redis.set('post:t3_abc123:views', '42');
await context.redis.set('subreddit:devvit:stats', JSON.stringify(stats));
await context.redis.set('app:settings:enabled', 'true');

// Use consistent separators (: is common)
// Format: namespace:entity:attribute
```

**Benefits:**
- Easy to understand
- Prevents key collisions
- Enables pattern matching (future feature)

## Common Patterns

### Pattern 1: Simple Counter

```typescript
Devvit.addMenuItem({
  label: 'Increment Counter',
  location: 'subreddit',
  onPress: async (event, context) => {
    const count = await context.redis.incrBy('global-counter', 1);
    context.ui.showToast(`Count: ${count}`);
  },
});
```

### Pattern 2: Per-User Data

```typescript
Devvit.addMenuItem({
  label: 'My Stats',
  location: 'subreddit',
  onPress: async (event, context) => {
    const user = await context.reddit.getCurrentUser();
    const key = `user:${user.id}:stats`;

    const statsStr = await context.redis.get(key);
    const stats = JSON.parse(statsStr || '{"clicks": 0}');

    stats.clicks += 1;
    await context.redis.set(key, JSON.stringify(stats));

    context.ui.showToast(`You've clicked ${stats.clicks} times`);
  },
});
```

### Pattern 3: Per-Post Data

```typescript
Devvit.addMenuItem({
  label: 'View Count',
  location: 'post',
  onPress: async (event, context) => {
    const key = `post:${event.targetId}:views`;
    const views = await context.redis.incrBy(key, 1);
    context.ui.showToast(`Views: ${views}`);
  },
});
```

### Pattern 4: Leaderboard

```typescript
interface LeaderboardEntry {
  username: string;
  score: number;
}

async function updateLeaderboard(
  context: Context,
  username: string,
  score: number
): Promise<void> {
  const leaderboardStr = await context.redis.get('leaderboard');
  const leaderboard: LeaderboardEntry[] = JSON.parse(leaderboardStr || '[]');

  // Update or add entry
  const existingIndex = leaderboard.findIndex(e => e.username === username);
  if (existingIndex >= 0) {
    leaderboard[existingIndex].score = score;
  } else {
    leaderboard.push({ username, score });
  }

  // Sort by score descending
  leaderboard.sort((a, b) => b.score - a.score);

  // Keep top 10
  const top10 = leaderboard.slice(0, 10);

  await context.redis.set('leaderboard', JSON.stringify(top10));
}

Devvit.addMenuItem({
  label: 'Show Leaderboard',
  location: 'subreddit',
  onPress: async (event, context) => {
    const leaderboardStr = await context.redis.get('leaderboard');
    const leaderboard: LeaderboardEntry[] = JSON.parse(leaderboardStr || '[]');

    const display = leaderboard
      .map((e, i) => `${i + 1}. ${e.username}: ${e.score}`)
      .join('\n');

    context.ui.showToast(display || 'Leaderboard is empty');
  },
});
```

### Pattern 5: Caching

```typescript
async function getCachedData(
  context: Context,
  key: string,
  fetchFn: () => Promise<any>,
  ttlMs: number = 3600000
): Promise<any> {
  // Try cache first
  const cached = await context.redis.get(key);
  if (cached) {
    return JSON.parse(cached);
  }

  // Cache miss, fetch fresh data
  const data = await fetchFn();

  // Store with expiration
  await context.redis.set(key, JSON.stringify(data), {
    expiration: new Date(Date.now() + ttlMs),
  });

  return data;
}

// Usage
Devvit.addMenuItem({
  label: 'Get Stats',
  location: 'subreddit',
  onPress: async (event, context) => {
    const stats = await getCachedData(
      context,
      'subreddit-stats',
      async () => {
        // Expensive operation
        const sub = await context.reddit.getCurrentSubreddit();
        return {
          subscribers: sub.numberOfSubscribers,
          fetchedAt: Date.now(),
        };
      },
      600000 // 10 minutes
    );

    context.ui.showToast(`Subscribers: ${stats.subscribers}`);
  },
});
```

### Pattern 6: Configuration Storage

```typescript
interface AppConfig {
  enabled: boolean;
  threshold: number;
  welcomeMessage: string;
}

const defaultConfig: AppConfig = {
  enabled: true,
  threshold: 10,
  welcomeMessage: 'Welcome!',
};

async function getConfig(context: Context): Promise<AppConfig> {
  const configStr = await context.redis.get('app:config');
  return configStr ? JSON.parse(configStr) : defaultConfig;
}

async function updateConfig(context: Context, config: AppConfig): Promise<void> {
  await context.redis.set('app:config', JSON.stringify(config));
}

Devvit.addMenuItem({
  label: 'Configure App',
  location: 'subreddit',
  forUserType: 'moderator',
  onPress: async (event, context) => {
    const config = await getConfig(context);

    const configForm = Devvit.createForm(
      {
        title: 'App Configuration',
        fields: [
          {
            name: 'enabled',
            label: 'Enabled',
            type: 'boolean',
            defaultValue: config.enabled,
          },
          {
            name: 'threshold',
            label: 'Threshold',
            type: 'number',
            defaultValue: config.threshold,
          },
          {
            name: 'welcomeMessage',
            label: 'Welcome Message',
            type: 'string',
            defaultValue: config.welcomeMessage,
          },
        ],
      },
      async (event, context) => {
        const newConfig: AppConfig = {
          enabled: event.values.enabled as boolean,
          threshold: event.values.threshold as number,
          welcomeMessage: event.values.welcomeMessage as string,
        };

        await updateConfig(context, newConfig);
        context.ui.showToast('Configuration updated!');
      }
    );

    context.ui.showForm(configForm);
  },
});
```

## Advanced Patterns

### Pattern 7: Rate Limiting

```typescript
async function isRateLimited(
  context: Context,
  userId: string,
  actionType: string,
  maxActions: number,
  windowMs: number
): Promise<boolean> {
  const key = `ratelimit:${userId}:${actionType}`;
  const countStr = await context.redis.get(key);
  const count = Number(countStr || 0);

  if (count >= maxActions) {
    return true; // Rate limited
  }

  if (count === 0) {
    // First action, set expiration
    await context.redis.set(key, '1', {
      expiration: new Date(Date.now() + windowMs),
    });
  } else {
    await context.redis.incrBy(key, 1);
  }

  return false;
}

Devvit.addMenuItem({
  label: 'Limited Action',
  location: 'post',
  onPress: async (event, context) => {
    const user = await context.reddit.getCurrentUser();

    const limited = await isRateLimited(
      context,
      user.id,
      'post-action',
      5,        // Max 5 actions
      60000     // Per minute
    );

    if (limited) {
      context.ui.showToast('Rate limited. Please wait.');
      return;
    }

    // Perform action
    context.ui.showToast('Action performed!');
  },
});
```

### Pattern 8: Session Management

```typescript
interface Session {
  userId: string;
  username: string;
  startedAt: number;
  data: any;
}

async function createSession(
  context: Context,
  userId: string,
  username: string
): Promise<string> {
  const sessionId = `session_${Date.now()}_${Math.random()}`;
  const session: Session = {
    userId,
    username,
    startedAt: Date.now(),
    data: {},
  };

  await context.redis.set(`session:${sessionId}`, JSON.stringify(session), {
    expiration: new Date(Date.now() + 3600000), // 1 hour
  });

  return sessionId;
}

async function getSession(
  context: Context,
  sessionId: string
): Promise<Session | null> {
  const sessionStr = await context.redis.get(`session:${sessionId}`);
  return sessionStr ? JSON.parse(sessionStr) : null;
}
```

### Pattern 9: Queue Implementation

```typescript
interface QueueItem {
  id: string;
  data: any;
  timestamp: number;
}

async function enqueue(context: Context, queueName: string, item: any): Promise<void> {
  const queueKey = `queue:${queueName}`;
  const queueStr = await context.redis.get(queueKey);
  const queue: QueueItem[] = JSON.parse(queueStr || '[]');

  queue.push({
    id: `${Date.now()}_${Math.random()}`,
    data: item,
    timestamp: Date.now(),
  });

  await context.redis.set(queueKey, JSON.stringify(queue));
}

async function dequeue(context: Context, queueName: string): Promise<QueueItem | null> {
  const queueKey = `queue:${queueName}`;
  const queueStr = await context.redis.get(queueKey);
  const queue: QueueItem[] = JSON.parse(queueStr || '[]');

  if (queue.length === 0) {
    return null;
  }

  const item = queue.shift()!;
  await context.redis.set(queueKey, JSON.stringify(queue));
  return item;
}

async function queueSize(context: Context, queueName: string): Promise<number> {
  const queueStr = await context.redis.get(`queue:${queueName}`);
  const queue: QueueItem[] = JSON.parse(queueStr || '[]');
  return queue.length;
}
```

## Complete Example: Vote Tracker

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
  redis: true,
});

interface VoteData {
  postId: string;
  upvotes: number;
  downvotes: number;
  voters: { [userId: string]: 'up' | 'down' };
}

async function getVoteData(context: Context, postId: string): Promise<VoteData> {
  const key = `votes:${postId}`;
  const dataStr = await context.redis.get(key);

  if (dataStr) {
    return JSON.parse(dataStr);
  }

  return {
    postId,
    upvotes: 0,
    downvotes: 0,
    voters: {},
  };
}

async function saveVoteData(context: Context, data: VoteData): Promise<void> {
  const key = `votes:${data.postId}`;
  await context.redis.set(key, JSON.stringify(data));
}

Devvit.addMenuItem({
  label: 'Upvote',
  location: 'post',
  onPress: async (event, context) => {
    const user = await context.reddit.getCurrentUser();
    const voteData = await getVoteData(context, event.targetId);

    const previousVote = voteData.voters[user.id];

    if (previousVote === 'up') {
      context.ui.showToast('Already upvoted');
      return;
    }

    if (previousVote === 'down') {
      voteData.downvotes--;
    }

    voteData.upvotes++;
    voteData.voters[user.id] = 'up';

    await saveVoteData(context, voteData);

    context.ui.showToast(`Upvoted! Total: ${voteData.upvotes}`);
  },
});

Devvit.addMenuItem({
  label: 'Downvote',
  location: 'post',
  onPress: async (event, context) => {
    const user = await context.reddit.getCurrentUser();
    const voteData = await getVoteData(context, event.targetId);

    const previousVote = voteData.voters[user.id];

    if (previousVote === 'down') {
      context.ui.showToast('Already downvoted');
      return;
    }

    if (previousVote === 'up') {
      voteData.upvotes--;
    }

    voteData.downvotes++;
    voteData.voters[user.id] = 'down';

    await saveVoteData(context, voteData);

    context.ui.showToast(`Downvoted! Total: ${voteData.downvotes}`);
  },
});

Devvit.addMenuItem({
  label: 'Show Votes',
  location: 'post',
  onPress: async (event, context) => {
    const voteData = await getVoteData(context, event.targetId);
    const total = voteData.upvotes - voteData.downvotes;

    context.ui.showToast(
      `⬆️ ${voteData.upvotes} ⬇️ ${voteData.downvotes} (${total})`
    );
  },
});

export default Devvit;
```

## Best Practices

### 1. Type Safety

Always type your stored data:

```typescript
interface StoredData {
  value: number;
  timestamp: number;
}

// Good: Type-safe
const data: StoredData = JSON.parse(dataStr || '{"value": 0, "timestamp": 0}');

// Bad: Any type
const data = JSON.parse(dataStr);
```

### 2. Default Values

Always provide defaults when parsing:

```typescript
// Good
const data = JSON.parse(dataStr || '[]');
const count = Number(countStr || '0');

// Bad: Can crash
const data = JSON.parse(dataStr); // Error if null!
```

### 3. Error Handling

Wrap Redis operations in try-catch:

```typescript
try {
  const data = await context.redis.get('key');
  // Process data
} catch (error) {
  console.error('Redis error:', error);
  // Handle gracefully
}
```

### 4. Atomic Operations

Use `incrBy` for counters (atomic):

```typescript
// Good: Atomic
const count = await context.redis.incrBy('counter', 1);

// Bad: Race condition
const count = Number(await context.redis.get('counter') || '0');
await context.redis.set('counter', String(count + 1));
```

### 5. Memory Management

Don't store unlimited data:

```typescript
// Bad: Array grows forever
const items = JSON.parse(await context.redis.get('items') || '[]');
items.push(newItem);
await context.redis.set('items', JSON.stringify(items));

// Good: Limit size
const items = JSON.parse(await context.redis.get('items') || '[]');
items.push(newItem);
const limited = items.slice(-100); // Keep last 100
await context.redis.set('items', JSON.stringify(limited));
```

### 6. Use Expiration

Set expiration for temporary data:

```typescript
// Cache with TTL
await context.redis.set('cache:key', data, {
  expiration: new Date(Date.now() + 3600000), // 1 hour
});
```

### 7. Consistent Key Naming

```typescript
// Good: Consistent pattern
const userKey = `user:${userId}:data`;
const postKey = `post:${postId}:stats`;

// Bad: Inconsistent
const userKey = `${userId}_data`;
const postKey = `post-${postId}-stats`;
```

## Debugging Storage Issues

### View stored data:

```typescript
Devvit.addMenuItem({
  label: 'Debug Storage',
  location: 'subreddit',
  forUserType: 'moderator',
  onPress: async (event, context) => {
    // List some keys and their values
    const keys = ['counter', 'config', 'leaderboard'];

    for (const key of keys) {
      const value = await context.redis.get(key);
      console.log(`${key}:`, value);
    }

    context.ui.showToast('Check console for storage data');
  },
});
```

### Clear all data:

```typescript
Devvit.addMenuItem({
  label: 'Reset App Data',
  location: 'subreddit',
  forUserType: 'moderator',
  onPress: async (event, context) => {
    const keysToDelete = ['counter', 'config', 'leaderboard'];
    await context.redis.del(keysToDelete);
    context.ui.showToast('App data reset');
  },
});
```

## Key Takeaways

1. **Redis is required** - For persistent state in stateless environment
2. **All values are strings** - Use JSON.stringify/parse for objects
3. **Use consistent key names** - namespace:entity:attribute pattern
4. **Atomic operations** - Use incrBy for counters
5. **Set expiration** - For temporary/cached data
6. **Error handling** - Always wrap in try-catch
7. **Type safety** - Define interfaces for stored data
8. **Memory management** - Limit array/object sizes

## Checkpoint Questions

1. What method stores a value in Redis?
2. How do you store an object in Redis?
3. What method atomically increments a counter?
4. How do you set a key to expire after 1 hour?
5. What's a good key naming pattern?

<details>
<summary>Click to see answers</summary>

1. `await context.redis.set('key', 'value')`
2. `await context.redis.set('key', JSON.stringify(object))`
3. `await context.redis.incrBy('counter', 1)`
4. `await context.redis.set('key', 'value', { expiration: new Date(Date.now() + 3600000) })`
5. `namespace:entity:attribute` (e.g., `user:alice:score`)

</details>

## Practice Exercises

### Exercise 1: User Points System

Create a system where users earn points for posting. Store and display their total points.

### Exercise 2: Daily Stats

Track daily post counts. Store stats per day and show a summary.

### Exercise 3: Favorites List

Let users save their favorite posts. Store per-user and display their list.

## Additional Resources

- [Redis Documentation](https://redis.io/docs/)
- [Devvit Storage Guide](https://developers.reddit.com/docs/storage)
- [Data Modeling with Redis](https://redis.io/topics/data-types-intro)

---

➡️ **Continue to [Lesson 8: Scheduler and Background Jobs](./08-scheduler.md)**

**Estimated time to complete**: 1.5 hours
**Practice exercises**: 3 suggested projects
**Prerequisites**: Lessons 1-6 completed
