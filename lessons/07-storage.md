# Lesson 7: Advanced Redis and Data Patterns

**Duration**: 2.5 hours
**Level**: Intermediate to Advanced

## Overview

Since Devvit functions are stateless, you need persistent storage to maintain data between executions. This lesson covers Redis storage comprehensively: from basics to advanced patterns including race conditions, transactions, data migrations, and production-ready strategies.

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

## Advanced Redis Patterns

### Race Conditions and Atomic Operations

Race conditions occur when multiple users or processes access the same data simultaneously.

#### Problem: Non-Atomic Updates

```typescript
// ❌ RACE CONDITION - Don't do this!
Devvit.addMenuItem({
  label: 'Claim Reward',
  location: 'post',
  onPress: async (event, context) => {
    // User A and User B both click at the same time
    const balance = Number(await context.redis.get('pool') || '1000');

    if (balance >= 100) {
      // Both see balance = 1000
      // Both pass the check
      await context.redis.set('pool', String(balance - 100));
      // Pool now = 900, but should be 800!

      context.ui.showToast('Claimed 100 coins!');
    }
  },
});
```

**What happens:**
1. User A reads balance: 1000
2. User B reads balance: 1000 (simultaneously)
3. User A writes: 900
4. User B writes: 900 (overwrites A's change!)
5. Result: 100 coins disappeared

#### Solution 1: Use Atomic Operations

```typescript
// ✅ ATOMIC - Safe from race conditions
Devvit.addMenuItem({
  label: 'Claim Reward (Safe)',
  location: 'post',
  onPress: async (event, context) => {
    try {
      // Atomic decrement
      const newBalance = await context.redis.incrBy('pool', -100);

      if (newBalance < 0) {
        // Oops, went negative, roll back
        await context.redis.incrBy('pool', 100);
        context.ui.showToast('Not enough coins in pool');
        return;
      }

      context.ui.showToast(`Claimed! Pool now at ${newBalance}`);
    } catch (error) {
      context.ui.showToast('Failed to claim');
    }
  },
});
```

#### Solution 2: Check-and-Set Pattern with Versioning

```typescript
interface VersionedData {
  value: number;
  version: number;
}

async function safeUpdate(
  context: Context,
  key: string,
  updateFn: (current: number) => number,
  maxRetries: number = 3
): Promise<boolean> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    // Get current value with version
    const dataStr = await context.redis.get(key);
    const data: VersionedData = dataStr
      ? JSON.parse(dataStr)
      : { value: 0, version: 0 };

    // Calculate new value
    const newValue = updateFn(data.value);
    const newData: VersionedData = {
      value: newValue,
      version: data.version + 1,
    };

    // Try to save with version check
    const versionKey = `${key}:version`;
    const currentVersion = await context.redis.get(versionKey);

    if (currentVersion === String(data.version)) {
      // Version matches, safe to update
      await context.redis.set(key, JSON.stringify(newData));
      await context.redis.set(versionKey, String(newData.version));
      return true;
    }

    // Version mismatch, someone else updated, retry
    console.log(`Version conflict on ${key}, retrying...`);
    await new Promise(resolve => setTimeout(resolve, 50 * attempt));
  }

  return false; // Failed after retries
}

// Usage
Devvit.addMenuItem({
  label: 'Safe Complex Update',
  location: 'post',
  onPress: async (event, context) => {
    const success = await safeUpdate(
      context,
      'complex-data',
      (current) => current + 10
    );

    if (success) {
      context.ui.showToast('Updated successfully');
    } else {
      context.ui.showToast('Update failed, too much contention');
    }
  },
});
```

#### Solution 3: Distributed Locks

```typescript
async function acquireLock(
  context: Context,
  lockKey: string,
  ttlMs: number = 5000
): Promise<boolean> {
  const lockValue = `${Date.now()}_${Math.random()}`;
  const acquired = await context.redis.set(lockKey, lockValue, {
    expiration: new Date(Date.now() + ttlMs),
  });

  // Check if we got the lock (should return null if key existed)
  const current = await context.redis.get(lockKey);
  return current === lockValue;
}

async function releaseLock(context: Context, lockKey: string): Promise<void> {
  await context.redis.del(lockKey);
}

async function withLock<T>(
  context: Context,
  lockKey: string,
  operation: () => Promise<T>,
  maxWaitMs: number = 3000
): Promise<T | null> {
  const start = Date.now();

  // Try to acquire lock
  while (Date.now() - start < maxWaitMs) {
    if (await acquireLock(context, lockKey)) {
      try {
        return await operation();
      } finally {
        await releaseLock(context, lockKey);
      }
    }

    // Wait before retry
    await new Promise(resolve => setTimeout(resolve, 100));
  }

  return null; // Couldn't acquire lock
}

// Usage
Devvit.addMenuItem({
  label: 'Locked Operation',
  location: 'post',
  onPress: async (event, context) => {
    const result = await withLock(
      context,
      `lock:post:${event.targetId}`,
      async () => {
        // Only one user can execute this at a time
        const data = await context.redis.get(`data:${event.targetId}`);
        const parsed = JSON.parse(data || '{"count": 0}');
        parsed.count++;
        await context.redis.set(`data:${event.targetId}`, JSON.stringify(parsed));
        return parsed.count;
      }
    );

    if (result !== null) {
      context.ui.showToast(`Count: ${result}`);
    } else {
      context.ui.showToast('Could not acquire lock');
    }
  },
});
```

### Data Migrations

As your app evolves, you'll need to migrate data to new schemas.

#### Migration Strategy 1: Versioned Data

```typescript
interface DataV1 {
  version: 1;
  username: string;
  score: number;
}

interface DataV2 {
  version: 2;
  username: string;
  score: number;
  achievements: string[];
  joinedAt: number;
}

type UserData = DataV1 | DataV2;

async function getUserData(context: Context, userId: string): Promise<DataV2> {
  const key = `user:${userId}`;
  const dataStr = await context.redis.get(key);

  if (!dataStr) {
    // New user, return v2 with defaults
    return {
      version: 2,
      username: 'Unknown',
      score: 0,
      achievements: [],
      joinedAt: Date.now(),
    };
  }

  const data = JSON.parse(dataStr) as UserData;

  // Migrate v1 to v2
  if (data.version === 1) {
    const migrated: DataV2 = {
      version: 2,
      username: data.username,
      score: data.score,
      achievements: [], // New field
      joinedAt: Date.now(), // New field
    };

    // Save migrated data
    await context.redis.set(key, JSON.stringify(migrated));
    console.log(`Migrated user ${userId} from v1 to v2`);

    return migrated;
  }

  return data as DataV2;
}
```

#### Migration Strategy 2: Background Migration Job

```typescript
// Scheduled job to migrate all users
Devvit.addSchedulerJob({
  name: 'migrateUsersToV2',
  cron: '0 2 * * *', // Run daily at 2 AM
  onRun: async (event, context) => {
    // Get list of all user keys (you'd need to track these)
    const userListStr = await context.redis.get('all-users');
    const userIds: string[] = JSON.parse(userListStr || '[]');

    let migrated = 0;
    let errors = 0;

    for (const userId of userIds) {
      try {
        const data = await getUserData(context, userId); // Uses migration logic above
        if (data.version === 2) {
          migrated++;
        }
      } catch (error) {
        console.error(`Failed to migrate user ${userId}:`, error);
        errors++;
      }

      // Rate limit: wait between migrations
      if (migrated % 10 === 0) {
        await new Promise(resolve => setTimeout(resolve, 100));
      }
    }

    console.log(`Migration complete: ${migrated} users migrated, ${errors} errors`);

    // Track migration progress
    await context.redis.set('migration:v2:complete', String(Date.now()));
  },
});
```

#### Migration Strategy 3: Lazy Migration

```typescript
// Migrate on read, write back on write
async function getLegacyCompatibleData(
  context: Context,
  key: string
): Promise<DataV2> {
  const dataStr = await context.redis.get(key);

  if (!dataStr) {
    return createDefaultV2Data();
  }

  const data = JSON.parse(dataStr);

  // Check for old schema (no version field)
  if (!data.version) {
    // This is v1 (before versioning)
    return {
      version: 2,
      username: data.name || 'Unknown', // Field name changed
      score: data.points || 0, // Field name changed
      achievements: [],
      joinedAt: Date.now(),
    };
  }

  if (data.version === 1) {
    return migrateV1ToV2(data);
  }

  return data;
}

async function saveData(context: Context, key: string, data: DataV2): Promise<void> {
  // Always save as v2
  await context.redis.set(key, JSON.stringify(data));
}
```

### Advanced Memory Management

#### Pattern 1: Sliding Window for Time-Series Data

```typescript
interface TimeSeriesEntry {
  timestamp: number;
  value: number;
}

async function addToTimeSeriesWindow(
  context: Context,
  key: string,
  value: number,
  windowMs: number = 3600000 // 1 hour
): Promise<void> {
  const now = Date.now();
  const dataStr = await context.redis.get(key);
  const entries: TimeSeriesEntry[] = dataStr ? JSON.parse(dataStr) : [];

  // Add new entry
  entries.push({ timestamp: now, value });

  // Remove entries outside window
  const cutoff = now - windowMs;
  const filtered = entries.filter(e => e.timestamp > cutoff);

  // Limit to last 1000 entries even within window
  const limited = filtered.slice(-1000);

  await context.redis.set(key, JSON.stringify(limited));
}

async function getTimeSeriesStats(
  context: Context,
  key: string
): Promise<{ count: number; sum: number; avg: number }> {
  const dataStr = await context.redis.get(key);
  const entries: TimeSeriesEntry[] = dataStr ? JSON.parse(dataStr) : [];

  const sum = entries.reduce((acc, e) => acc + e.value, 0);

  return {
    count: entries.length,
    sum,
    avg: entries.length > 0 ? sum / entries.length : 0,
  };
}

// Usage: Track post views over last hour
Devvit.addMenuItem({
  label: 'View Stats (Last Hour)',
  location: 'post',
  onPress: async (event, context) => {
    const key = `views:${event.targetId}`;

    // Record this view
    await addToTimeSeriesWindow(context, key, 1);

    // Get stats
    const stats = await getTimeSeriesStats(context, key);

    context.ui.showToast(
      `Views (last hour): ${stats.count}\nAvg: ${stats.avg.toFixed(2)}`
    );
  },
});
```

#### Pattern 2: Circular Buffer

```typescript
interface CircularBuffer<T> {
  items: T[];
  maxSize: number;
  nextIndex: number;
}

async function addToCircularBuffer<T>(
  context: Context,
  key: string,
  item: T,
  maxSize: number = 100
): Promise<void> {
  const dataStr = await context.redis.get(key);
  const buffer: CircularBuffer<T> = dataStr
    ? JSON.parse(dataStr)
    : { items: [], maxSize, nextIndex: 0 };

  // Update max size if changed
  buffer.maxSize = maxSize;

  // Add item at next index
  if (buffer.items.length < buffer.maxSize) {
    buffer.items.push(item);
  } else {
    buffer.items[buffer.nextIndex] = item;
  }

  // Advance index (wrap around)
  buffer.nextIndex = (buffer.nextIndex + 1) % buffer.maxSize;

  await context.redis.set(key, JSON.stringify(buffer));
}

async function getCircularBufferItems<T>(
  context: Context,
  key: string
): Promise<T[]> {
  const dataStr = await context.redis.get(key);
  if (!dataStr) return [];

  const buffer: CircularBuffer<T> = JSON.parse(dataStr);

  // Return items in chronological order
  const { items, nextIndex, maxSize } = buffer;

  if (items.length < maxSize) {
    return items;
  }

  // Reorder: items from nextIndex to end, then start to nextIndex
  return [...items.slice(nextIndex), ...items.slice(0, nextIndex)];
}

// Usage: Keep last 50 errors
async function logError(context: Context, error: Error): Promise<void> {
  await addToCircularBuffer(
    context,
    'app:recent-errors',
    {
      message: error.message,
      stack: error.stack,
      timestamp: Date.now(),
    },
    50
  );
}
```

#### Pattern 3: Data Compression

```typescript
// For large datasets, compress before storing
async function setCompressed(
  context: Context,
  key: string,
  data: any
): Promise<void> {
  const json = JSON.stringify(data);

  // Simple compression: remove whitespace
  const compressed = json;

  // For real compression, you'd use a library (if available)
  // const compressed = compress(json);

  await context.redis.set(key, compressed);
}

async function getDecompressed<T>(context: Context, key: string): Promise<T | null> {
  const compressed = await context.redis.get(key);
  if (!compressed) return null;

  // const json = decompress(compressed);
  const json = compressed;

  return JSON.parse(json);
}

// Pattern 4: Sampling for high-frequency data
async function sampleAndStore(
  context: Context,
  key: string,
  value: number,
  sampleRate: number = 0.1 // Keep 10% of data
): Promise<void> {
  if (Math.random() < sampleRate) {
    await addToTimeSeriesWindow(context, key, value);
  }
}
```

### Data Backup and Recovery

#### Pattern 1: Periodic Snapshots

```typescript
interface Snapshot {
  timestamp: number;
  keys: { [key: string]: string };
}

async function createSnapshot(context: Context, keyPattern: string): Promise<string> {
  // In production, you'd need a way to list keys by pattern
  // For this example, we track keys manually
  const keyListStr = await context.redis.get('tracked-keys');
  const keys: string[] = JSON.parse(keyListStr || '[]');

  const snapshot: Snapshot = {
    timestamp: Date.now(),
    keys: {},
  };

  for (const key of keys) {
    const value = await context.redis.get(key);
    if (value) {
      snapshot.keys[key] = value;
    }
  }

  const snapshotId = `snapshot:${Date.now()}`;
  await context.redis.set(snapshotId, JSON.stringify(snapshot), {
    expiration: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 days
  });

  return snapshotId;
}

async function restoreFromSnapshot(
  context: Context,
  snapshotId: string
): Promise<number> {
  const snapshotStr = await context.redis.get(snapshotId);
  if (!snapshotStr) {
    throw new Error('Snapshot not found');
  }

  const snapshot: Snapshot = JSON.parse(snapshotStr);
  let restored = 0;

  for (const [key, value] of Object.entries(snapshot.keys)) {
    await context.redis.set(key, value);
    restored++;
  }

  return restored;
}

// Scheduled backup
Devvit.addSchedulerJob({
  name: 'dailyBackup',
  cron: '0 3 * * *', // 3 AM daily
  onEvent: async (event, context) => {
    const snapshotId = await createSnapshot(context, 'app:*');
    console.log(`Backup created: ${snapshotId}`);

    await context.redis.set('last-backup-id', snapshotId);
  },
});
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

**Estimated time to complete**: 2.5 hours
**Practice exercises**: 3 suggested projects
**Prerequisites**: Lessons 1-6 completed

**What you learned:**
- Basic and advanced Redis operations
- Race condition handling and atomic operations
- Distributed locks
- Data migrations strategies
- Advanced memory management patterns
- Backup and recovery
