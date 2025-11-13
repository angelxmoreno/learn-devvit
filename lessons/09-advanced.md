# Lesson 9: Advanced Features and Real-time Patterns

**Duration**: 3 hours
**Level**: Advanced

## Overview

Master advanced Devvit features including HTTP requests, media handling, app settings, real-time patterns, game architecture, and production-ready best practices. This lesson covers the cutting edge of what's possible with Devvit.

## HTTP Requests

Make requests to external APIs (with restrictions).

### Enabling HTTP

```typescript
Devvit.configure({
  redditAPI: true,
  http: true,  // Enable HTTP fetch
});
```

### Basic Fetch

```typescript
const response = await fetch('https://api.example.com/data');
const data = await response.json();

console.log(data);
```

### Complete Example

```typescript
Devvit.addMenuItem({
  label: 'Fetch Weather',
  location: 'subreddit',
  onPress: async (event, context) => {
    try {
      const response = await fetch('https://api.weather.com/v1/current?city=NYC');

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const data = await response.json();

      context.ui.showToast(`Weather: ${data.temperature}°F`);
    } catch (error) {
      console.error('Fetch error:', error);
      context.ui.showToast({
        text: 'Failed to fetch weather',
        appearance: 'error',
      });
    }
  },
});
```

### POST Requests

```typescript
const response = await fetch('https://api.example.com/submit', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    title: 'My Data',
    value: 42,
  }),
});

const result = await response.json();
```

### With Authentication

```typescript
const response = await fetch('https://api.example.com/protected', {
  headers: {
    'Authorization': `Bearer ${apiKey}`,
    'Content-Type': 'application/json',
  },
});
```

### HTTP Best Practices

1. **Always handle errors**:
```typescript
try {
  const response = await fetch(url);
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);
  }
  const data = await response.json();
} catch (error) {
  console.error('Fetch failed:', error);
  // Handle gracefully
}
```

2. **Set timeouts** (via AbortController):
```typescript
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 5000); // 5 seconds

try {
  const response = await fetch(url, { signal: controller.signal });
  clearTimeout(timeout);
  // Process response
} catch (error) {
  if (error.name === 'AbortError') {
    console.log('Request timed out');
  }
}
```

3. **Cache responses**:
```typescript
async function cachedFetch(
  context: Context,
  url: string,
  cacheKey: string,
  ttlMs: number = 600000
): Promise<any> {
  // Check cache
  const cached = await context.redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // Fetch fresh data
  const response = await fetch(url);
  const data = await response.json();

  // Cache it
  await context.redis.set(cacheKey, JSON.stringify(data), {
    expiration: new Date(Date.now() + ttlMs),
  });

  return data;
}
```

## App Settings

Create user-configurable settings for your app.

### Define Settings in devvit.yaml

```yaml
name: my-app
version: 1.0.0
settings:
  - name: apiKey
    type: string
    label: API Key
    description: Your external API key
    isSecret: true

  - name: threshold
    type: number
    label: Score Threshold
    description: Minimum score for actions
    defaultValue: 10

  - name: enabled
    type: boolean
    label: Enable Feature
    description: Turn feature on/off
    defaultValue: true

  - name: allowedUsers
    type: string
    label: Allowed Users
    description: Comma-separated list of usernames
```

### Setting Types

- **string** - Text input (use `isSecret: true` for passwords/keys)
- **number** - Numeric input
- **boolean** - True/false toggle
- **select** - Dropdown (define options in settings)

### Access Settings in Code

```typescript
Devvit.addMenuItem({
  label: 'Check Settings',
  location: 'subreddit',
  onPress: async (event, context) => {
    const apiKey = await context.settings.get('apiKey');
    const threshold = await context.settings.get('threshold');
    const enabled = await context.settings.get('enabled');
    const allowedUsers = await context.settings.get('allowedUsers');

    console.log('API Key:', apiKey ? '***' : 'Not set');
    console.log('Threshold:', threshold);
    console.log('Enabled:', enabled);
    console.log('Allowed users:', allowedUsers);

    if (!enabled) {
      context.ui.showToast('Feature is disabled');
      return;
    }

    // Use settings
    if (threshold && typeof threshold === 'number') {
      // Apply threshold logic
    }
  },
});
```

### Settings Example

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
  http: true,
});

Devvit.addTrigger({
  event: 'PostSubmit',
  onEvent: async (event, context) => {
    // Check if feature is enabled
    const enabled = await context.settings.get('enabled');
    if (!enabled) return;

    // Get threshold
    const threshold = (await context.settings.get('threshold')) as number || 10;

    // Get API key
    const apiKey = await context.settings.get('apiKey') as string;
    if (!apiKey) {
      console.error('API key not configured');
      return;
    }

    // Use settings
    const post = event.post;

    if (post.score >= threshold) {
      // Call external API
      const response = await fetch('https://api.example.com/analyze', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${apiKey}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          title: post.title,
          url: post.url,
        }),
      });

      const result = await response.json();
      console.log('Analysis result:', result);
    }
  },
});

export default Devvit;
```

## Media Handling

Upload and process images.

### Enabling Media

```typescript
Devvit.configure({
  redditAPI: true,
  media: true,  // Enable media
});
```

### Upload Image

```typescript
// From URL
const mediaUrl = await context.media.upload({
  url: 'https://example.com/image.jpg',
  type: 'image',
});

// From base64
const mediaUrl = await context.media.upload({
  data: base64ImageData,
  type: 'image',
});

console.log('Uploaded media URL:', mediaUrl);
```

### Use in Posts

```typescript
const imageUrl = await context.media.upload({
  url: 'https://example.com/chart.png',
  type: 'image',
});

const post = await context.reddit.submitPost({
  title: 'Check out this chart',
  subredditName: 'test',
  url: imageUrl,  // Link post with uploaded image
});
```

### Use in Custom Posts

```typescript
Devvit.addCustomPostType({
  name: 'image-post',
  height: 'tall',
  render: (context) => {
    return (
      <vstack padding="medium" alignment="center middle">
        <image
          url="https://i.redd.it/some-image.jpg"
          imageWidth={400}
          imageHeight={300}
          description="My image"
        />
      </vstack>
    );
  },
});
```

## Performance Optimization

### 1. Parallel Requests

```typescript
// Bad: Sequential (slow)
const post = await context.reddit.getPostById(postId);
const author = await context.reddit.getUserById(post.authorId);
const subreddit = await context.reddit.getSubredditById(post.subredditId);

// Good: Parallel (fast)
const post = await context.reddit.getPostById(postId);
const [author, subreddit] = await Promise.all([
  context.reddit.getUserById(post.authorId),
  context.reddit.getSubredditById(post.subredditId),
]);
```

### 2. Batch Operations

```typescript
// Bad: Loop with await
for (const item of items) {
  await processItem(item);
}

// Good: Batch with Promise.all
await Promise.all(items.map(item => processItem(item)));

// Better: Chunked batches (avoid overwhelming)
const chunkSize = 10;
for (let i = 0; i < items.length; i += chunkSize) {
  const chunk = items.slice(i, i + chunkSize);
  await Promise.all(chunk.map(item => processItem(item)));
}
```

### 3. Caching

```typescript
// Cache expensive operations
async function getExpensiveData(context: Context): Promise<any> {
  const cacheKey = 'expensive-data';
  const cached = await context.redis.get(cacheKey);

  if (cached) {
    return JSON.parse(cached);
  }

  const data = await performExpensiveOperation();

  await context.redis.set(cacheKey, JSON.stringify(data), {
    expiration: new Date(Date.now() + 600000), // 10 minutes
  });

  return data;
}
```

### 4. Early Returns

```typescript
// Exit early when possible
Devvit.addMenuItem({
  label: 'Process Post',
  location: 'post',
  onPress: async (event, context) => {
    const user = await context.reddit.getCurrentUser();
    const subreddit = await context.reddit.getCurrentSubreddit();

    // Check permissions first
    const isMod = await user.isModerator(subreddit.name);
    if (!isMod) {
      context.ui.showToast('Moderators only');
      return; // Exit early
    }

    // Only fetch post if user is authorized
    const post = await context.reddit.getPostById(event.targetId);

    // Continue processing...
  },
});
```

### 5. Debouncing with Redis

```typescript
async function debounce(
  context: Context,
  key: string,
  delayMs: number
): Promise<boolean> {
  const lastRun = await context.redis.get(key);

  if (lastRun) {
    const elapsed = Date.now() - Number(lastRun);
    if (elapsed < delayMs) {
      return false; // Too soon
    }
  }

  await context.redis.set(key, String(Date.now()));
  return true; // OK to proceed
}

Devvit.addMenuItem({
  label: 'Rate Limited Action',
  location: 'post',
  onPress: async (event, context) => {
    const user = await context.reddit.getCurrentUser();
    const canProceed = await debounce(context, `debounce:${user.id}`, 5000);

    if (!canProceed) {
      context.ui.showToast('Please wait before trying again');
      return;
    }

    // Perform action
  },
});
```

## Security Best Practices

### 1. Validate User Input

```typescript
const form = Devvit.createForm(
  {
    fields: [
      { name: 'url', label: 'URL', type: 'string' },
    ],
  },
  async (event, context) => {
    const url = event.values.url as string;

    // Validate URL
    try {
      const parsed = new URL(url);

      if (!['http:', 'https:'].includes(parsed.protocol)) {
        context.ui.showToast('Invalid protocol');
        return;
      }
    } catch {
      context.ui.showToast('Invalid URL');
      return;
    }

    // Use validated URL
  }
);
```

### 2. Sanitize Data

```typescript
function sanitizeText(text: string): string {
  // Remove potential harmful characters
  return text
    .replace(/[<>]/g, '')  // Remove < and >
    .trim()
    .slice(0, 500);  // Limit length
}

const userInput = sanitizeText(event.values.message as string);
```

### 3. Check Permissions

```typescript
async function ensureModerator(context: Context): Promise<boolean> {
  const user = await context.reddit.getCurrentUser();
  const subreddit = await context.reddit.getCurrentSubreddit();
  return await user.isModerator(subreddit.name);
}

Devvit.addMenuItem({
  label: 'Mod Action',
  location: 'post',
  onPress: async (event, context) => {
    if (!(await ensureModerator(context))) {
      context.ui.showToast('Unauthorized');
      return;
    }

    // Perform mod action
  },
});
```

### 4. Rate Limiting

```typescript
async function checkRateLimit(
  context: Context,
  userId: string,
  maxRequests: number,
  windowMs: number
): Promise<{ allowed: boolean; remaining: number }> {
  const key = `ratelimit:${userId}`;
  const count = await context.redis.get(key);
  const current = Number(count || 0);

  if (current >= maxRequests) {
    return { allowed: false, remaining: 0 };
  }

  if (current === 0) {
    await context.redis.set(key, '1', {
      expiration: new Date(Date.now() + windowMs),
    });
  } else {
    await context.redis.incrBy(key, 1);
  }

  return {
    allowed: true,
    remaining: maxRequests - current - 1,
  };
}
```

### 5. Secrets Management

```typescript
// Use app settings for secrets (marked as isSecret: true in devvit.yaml)
const apiKey = await context.settings.get('apiKey');

// Never log secrets
console.log('API Key:', apiKey ? '***' : 'Not set');  // Good
console.log('API Key:', apiKey);  // Bad!

// Never store secrets in Redis without encryption
// Bad:
await context.redis.set('api-key', apiKey);  // Don't do this!
```

## Error Handling Patterns

### 1. Comprehensive Try-Catch

```typescript
Devvit.addMenuItem({
  label: 'Safe Action',
  location: 'post',
  onPress: async (event, context) => {
    try {
      const post = await context.reddit.getPostById(event.targetId);

      if (!post) {
        context.ui.showToast('Post not found');
        return;
      }

      // Process post
      await processPost(post);

      context.ui.showToast({
        text: 'Success!',
        appearance: 'success',
      });
    } catch (error) {
      console.error('Error in action:', error);

      // User-friendly error message
      context.ui.showToast({
        text: 'Something went wrong. Please try again.',
        appearance: 'error',
      });

      // Log detailed error for debugging
      await context.redis.set('last-error', JSON.stringify({
        timestamp: Date.now(),
        error: String(error),
        stack: error instanceof Error ? error.stack : undefined,
      }));
    }
  },
});
```

### 2. Retry Logic

```typescript
async function fetchWithRetry(
  url: string,
  maxRetries: number = 3,
  delayMs: number = 1000
): Promise<Response> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await fetch(url);

      if (response.ok) {
        return response;
      }

      // If server error, retry
      if (response.status >= 500) {
        console.log(`Attempt ${i + 1} failed, retrying...`);
        await new Promise(resolve => setTimeout(resolve, delayMs * (i + 1)));
        continue;
      }

      // Client error, don't retry
      throw new Error(`HTTP ${response.status}`);
    } catch (error) {
      if (i === maxRetries - 1) {
        throw error;
      }
      await new Promise(resolve => setTimeout(resolve, delayMs * (i + 1)));
    }
  }

  throw new Error('Max retries exceeded');
}
```

### 3. Graceful Degradation

```typescript
Devvit.addMenuItem({
  label: 'Show Stats',
  location: 'subreddit',
  onPress: async (event, context) => {
    try {
      // Try to get live data
      const stats = await fetchLiveStats(context);
      context.ui.showToast(`Live stats: ${stats.count}`);
    } catch (error) {
      console.error('Live stats failed:', error);

      // Fall back to cached data
      try {
        const cached = await context.redis.get('cached-stats');
        if (cached) {
          const stats = JSON.parse(cached);
          context.ui.showToast(`Cached stats: ${stats.count} (may be outdated)`);
        } else {
          context.ui.showToast('Stats unavailable');
        }
      } catch {
        context.ui.showToast('Stats unavailable');
      }
    }
  },
});
```

## Code Organization

### Project Structure

```
src/
├── main.tsx              # Entry point
├── handlers/
│   ├── postHandlers.ts   # Post-related handlers
│   ├── commentHandlers.ts
│   └── modHandlers.ts
├── services/
│   ├── reddit.ts         # Reddit API wrappers
│   ├── storage.ts        # Redis operations
│   └── external.ts       # External API calls
├── utils/
│   ├── formatting.ts     # Format helpers
│   ├── validation.ts     # Input validation
│   └── constants.ts      # Constants
├── types/
│   └── index.ts          # TypeScript types
└── components/
    └── customPosts.tsx   # Custom post components
```

### Type Definitions

```typescript
// types/index.ts
export interface AppConfig {
  enabled: boolean;
  threshold: number;
  message: string;
}

export interface UserStats {
  username: string;
  posts: number;
  comments: number;
  score: number;
}

export interface StorageKeys {
  CONFIG: 'app:config';
  USER_STATS: (userId: string) => `user:${string}:stats`;
  POST_DATA: (postId: string) => `post:${string}:data`;
}
```

### Service Layer

```typescript
// services/storage.ts
import { Context } from '@devvit/public-api';
import { AppConfig, StorageKeys } from '../types/index.js';

const KEYS: StorageKeys = {
  CONFIG: 'app:config',
  USER_STATS: (userId: string) => `user:${userId}:stats`,
  POST_DATA: (postId: string) => `post:${postId}:data`,
};

export async function getConfig(context: Context): Promise<AppConfig | null> {
  const data = await context.redis.get(KEYS.CONFIG);
  return data ? JSON.parse(data) : null;
}

export async function setConfig(context: Context, config: AppConfig): Promise<void> {
  await context.redis.set(KEYS.CONFIG, JSON.stringify(config));
}
```

### Reusable Handlers

```typescript
// handlers/postHandlers.ts
import { Context, MenuItemOnPressEvent } from '@devvit/public-api';

export async function handlePostAnalysis(
  event: MenuItemOnPressEvent,
  context: Context
): Promise<void> {
  try {
    const post = await context.reddit.getPostById(event.targetId);

    // Analysis logic
    const analysis = await analyzePost(post);

    context.ui.showToast(`Analysis: ${analysis.summary}`);
  } catch (error) {
    console.error('Analysis failed:', error);
    context.ui.showToast({
      text: 'Analysis failed',
      appearance: 'error',
    });
  }
}

async function analyzePost(post: any): Promise<{ summary: string }> {
  // Implementation
  return { summary: 'Post looks good' };
}
```

### Main Entry Point

```typescript
// main.tsx
import { Devvit } from '@devvit/public-api';
import { handlePostAnalysis } from './handlers/postHandlers.js';

Devvit.configure({
  redditAPI: true,
  redis: true,
});

Devvit.addMenuItem({
  label: 'Analyze Post',
  location: 'post',
  onPress: handlePostAnalysis,
});

export default Devvit;
```

## Testing Patterns

### Mock Data

```typescript
// For development/testing
const MOCK_MODE = false;

async function getPostData(context: Context, postId: string) {
  if (MOCK_MODE) {
    return {
      id: postId,
      title: 'Mock Post',
      score: 100,
    };
  }

  const post = await context.reddit.getPostById(postId);
  return {
    id: post.id,
    title: post.title,
    score: post.score,
  };
}
```

### Debug Logging

```typescript
const DEBUG = true;

function debugLog(...args: any[]) {
  if (DEBUG) {
    console.log('[DEBUG]', ...args);
  }
}

Devvit.addMenuItem({
  label: 'Action',
  location: 'post',
  onPress: async (event, context) => {
    debugLog('Handler started', event.targetId);

    const post = await context.reddit.getPostById(event.targetId);
    debugLog('Post fetched:', post.title);

    // Continue...
  },
});
```

## Real-time Features and Patterns

While Devvit doesn't support traditional WebSockets, you can implement real-time-like experiences using polling, Redis pub/sub simulation, and smart state management.

### Pattern 1: Polling-Based Real-time

```typescript
// In a custom post, poll for updates
Devvit.addCustomPostType({
  name: 'live-counter',
  height: 'regular',
  render: (context) => {
    const [count, setCount] = context.useState(0);
    const [isPolling, setIsPolling] = context.useState(true);

    // Poll every 2 seconds
    context.useInterval(async () => {
      if (!isPolling) return;

      const current = await context.redis.get('global-counter');
      const newCount = Number(current || 0);

      if (newCount !== count) {
        setCount(newCount);
      }
    }, 2000);

    const increment = async () => {
      const newCount = await context.redis.incrBy('global-counter', 1);
      setCount(newCount);
    };

    return (
      <vstack padding="medium" alignment="center middle" gap="medium">
        <text size="xxlarge" weight="bold">
          {count}
        </text>
        <text size="small" color="neutral-content-weak">
          Global counter (live updates)
        </text>
        <button onPress={increment}>Increment</button>
        <button
          onPress={() => setIsPolling(!isPolling)}
          appearance="secondary"
          size="small"
        >
          {isPolling ? 'Pause Updates' : 'Resume Updates'}
        </button>
      </vstack>
    );
  },
});
```

### Pattern 2: Event Broadcasting via Redis

```typescript
interface BroadcastMessage {
  type: string;
  payload: any;
  timestamp: number;
  sender: string;
}

async function broadcast(
  context: Context,
  channel: string,
  type: string,
  payload: any
): Promise<void> {
  const user = await context.reddit.getCurrentUser();

  const message: BroadcastMessage = {
    type,
    payload,
    timestamp: Date.now(),
    sender: user.username,
  };

  // Store message in a list
  const key = `channel:${channel}:messages`;
  const messagesStr = await context.redis.get(key);
  const messages: BroadcastMessage[] = messagesStr ? JSON.parse(messagesStr) : [];

  messages.push(message);

  // Keep only last 100 messages
  const limited = messages.slice(-100);

  await context.redis.set(key, JSON.stringify(limited), {
    expiration: new Date(Date.now() + 3600000), // 1 hour TTL
  });

  // Update last message timestamp for channel
  await context.redis.set(`channel:${channel}:last-update`, String(Date.now()));
}

async function getNewMessages(
  context: Context,
  channel: string,
  since: number
): Promise<BroadcastMessage[]> {
  const key = `channel:${channel}:messages`;
  const messagesStr = await context.redis.get(key);
  const messages: BroadcastMessage[] = messagesStr ? JSON.parse(messagesStr) : [];

  return messages.filter(m => m.timestamp > since);
}

// Usage in custom post
Devvit.addCustomPostType({
  name: 'chat-room',
  height: 'tall',
  render: (context) => {
    const [messages, setMessages] = context.useState<BroadcastMessage[]>([]);
    const [lastCheck, setLastCheck] = context.useState(Date.now());

    // Poll for new messages
    context.useInterval(async () => {
      const newMessages = await getNewMessages(context, 'global-chat', lastCheck);

      if (newMessages.length > 0) {
        setMessages([...messages, ...newMessages]);
        setLastCheck(Date.now());
      }
    }, 1000);

    const sendMessage = async (text: string) => {
      await broadcast(context, 'global-chat', 'message', { text });
    };

    return (
      <vstack padding="medium" gap="small">
        <text size="large" weight="bold">Live Chat</text>

        <vstack
          height="300px"
          backgroundColor="neutral-background-weak"
          cornerRadius="medium"
          padding="small"
          gap="small"
        >
          {messages.map((msg, i) => (
            <hstack key={i} gap="small">
              <text weight="bold">{msg.sender}:</text>
              <text>{msg.payload.text}</text>
            </hstack>
          ))}
        </vstack>

        <button onPress={() => {
          // Would show a form to get message text
          sendMessage('Hello!');
        }}>
          Send Message
        </button>
      </vstack>
    );
  },
});
```

### Pattern 3: Presence Detection

```typescript
interface PresenceInfo {
  userId: string;
  username: string;
  lastSeen: number;
}

async function updatePresence(context: Context, location: string): Promise<void> {
  const user = await context.reddit.getCurrentUser();

  const presenceKey = `presence:${location}`;
  const presenceStr = await context.redis.get(presenceKey);
  const presence: { [userId: string]: PresenceInfo } = presenceStr
    ? JSON.parse(presenceStr)
    : {};

  presence[user.id] = {
    userId: user.id,
    username: user.username,
    lastSeen: Date.now(),
  };

  await context.redis.set(presenceKey, JSON.stringify(presence), {
    expiration: new Date(Date.now() + 300000), // 5 min TTL
  });
}

async function getOnlineUsers(
  context: Context,
  location: string,
  timeoutMs: number = 60000
): Promise<PresenceInfo[]> {
  const presenceKey = `presence:${location}`;
  const presenceStr = await context.redis.get(presenceKey);

  if (!presenceStr) return [];

  const presence: { [userId: string]: PresenceInfo } = JSON.parse(presenceStr);
  const cutoff = Date.now() - timeoutMs;

  return Object.values(presence).filter(p => p.lastSeen > cutoff);
}

// Usage
Devvit.addCustomPostType({
  name: 'online-users',
  height: 'regular',
  render: (context) => {
    const [onlineUsers, setOnlineUsers] = context.useState<PresenceInfo[]>([]);

    // Update own presence every 30 seconds
    context.useInterval(async () => {
      await updatePresence(context, context.postId!);
    }, 30000);

    // Check for online users every 10 seconds
    context.useInterval(async () => {
      const users = await getOnlineUsers(context, context.postId!);
      setOnlineUsers(users);
    }, 10000);

    return (
      <vstack padding="medium" gap="small">
        <text size="large" weight="bold">
          Online Users ({onlineUsers.length})
        </text>
        {onlineUsers.map(user => (
          <text key={user.userId}>• u/{user.username}</text>
        ))}
      </vstack>
    );
  },
});
```

## Game Architecture Patterns

Building games in Devvit requires special patterns to handle state, turns, and multiplayer.

### Pattern 1: Turn-Based Game State

```typescript
interface GameState {
  gameId: string;
  players: Player[];
  currentPlayerIndex: number;
  board: any; // Game-specific board state
  status: 'waiting' | 'in-progress' | 'finished';
  winner?: string;
  createdAt: number;
  lastMove: number;
}

interface Player {
  userId: string;
  username: string;
  score: number;
}

async function createGame(context: Context, gameId: string): Promise<GameState> {
  const initialState: GameState = {
    gameId,
    players: [],
    currentPlayerIndex: 0,
    board: initializeBoard(), // Game-specific
    status: 'waiting',
    createdAt: Date.now(),
    lastMove: Date.now(),
  };

  await context.redis.set(`game:${gameId}`, JSON.stringify(initialState));
  return initialState;
}

async function joinGame(
  context: Context,
  gameId: string,
  userId: string,
  username: string
): Promise<GameState | null> {
  const gameStr = await context.redis.get(`game:${gameId}`);
  if (!gameStr) return null;

  const game: GameState = JSON.parse(gameStr);

  // Check if already joined
  if (game.players.some(p => p.userId === userId)) {
    return game;
  }

  // Check if game is full
  if (game.players.length >= 2) {
    return null;
  }

  game.players.push({ userId, username, score: 0 });

  // Start game when 2 players joined
  if (game.players.length === 2) {
    game.status = 'in-progress';
  }

  await context.redis.set(`game:${gameId}`, JSON.stringify(game));
  return game;
}

async function makeMove(
  context: Context,
  gameId: string,
  userId: string,
  move: any
): Promise<{ success: boolean; message: string; newState?: GameState }> {
  const gameStr = await context.redis.get(`game:${gameId}`);
  if (!gameStr) {
    return { success: false, message: 'Game not found' };
  }

  const game: GameState = JSON.parse(gameStr);

  // Validate it's player's turn
  const currentPlayer = game.players[game.currentPlayerIndex];
  if (currentPlayer.userId !== userId) {
    return { success: false, message: 'Not your turn!' };
  }

  // Validate and apply move (game-specific logic)
  const moveResult = applyMove(game, move);
  if (!moveResult.valid) {
    return { success: false, message: moveResult.error || 'Invalid move' };
  }

  // Update game state
  game.board = moveResult.newBoard;
  game.lastMove = Date.now();

  // Check for win condition
  const winCheck = checkWinCondition(game);
  if (winCheck.hasWinner) {
    game.status = 'finished';
    game.winner = winCheck.winner;
  } else {
    // Next player's turn
    game.currentPlayerIndex = (game.currentPlayerIndex + 1) % game.players.length;
  }

  await context.redis.set(`game:${gameId}`, JSON.stringify(game));

  return { success: true, message: 'Move made', newState: game };
}

// Game-specific functions (implement based on your game)
function initializeBoard(): any {
  return {}; // e.g., chess board, tic-tac-toe grid
}

function applyMove(game: GameState, move: any): { valid: boolean; newBoard?: any; error?: string } {
  // Validate and apply move to board
  return { valid: true, newBoard: game.board };
}

function checkWinCondition(game: GameState): { hasWinner: boolean; winner?: string } {
  // Check if someone won
  return { hasWinner: false };
}
```

### Pattern 2: Leaderboard with Rankings

```typescript
interface LeaderboardEntry {
  userId: string;
  username: string;
  score: number;
  wins: number;
  losses: number;
  lastPlayed: number;
}

async function updateLeaderboard(
  context: Context,
  userId: string,
  username: string,
  scoreChange: number,
  won: boolean
): Promise<void> {
  const leaderboardKey = 'game:leaderboard';
  const leaderboardStr = await context.redis.get(leaderboardKey);
  const leaderboard: LeaderboardEntry[] = leaderboardStr
    ? JSON.parse(leaderboardStr)
    : [];

  // Find or create entry
  let entry = leaderboard.find(e => e.userId === userId);

  if (!entry) {
    entry = {
      userId,
      username,
      score: 0,
      wins: 0,
      losses: 0,
      lastPlayed: Date.now(),
    };
    leaderboard.push(entry);
  }

  // Update stats
  entry.score += scoreChange;
  entry.lastPlayed = Date.now();

  if (won) {
    entry.wins++;
  } else {
    entry.losses++;
  }

  // Sort by score
  leaderboard.sort((a, b) => b.score - a.score);

  // Keep top 100
  const top100 = leaderboard.slice(0, 100);

  await context.redis.set(leaderboardKey, JSON.stringify(top100));
}

async function getLeaderboard(
  context: Context,
  limit: number = 10
): Promise<LeaderboardEntry[]> {
  const leaderboardStr = await context.redis.get('game:leaderboard');
  const leaderboard: LeaderboardEntry[] = leaderboardStr
    ? JSON.parse(leaderboardStr)
    : [];

  return leaderboard.slice(0, limit);
}

async function getUserRank(context: Context, userId: string): Promise<number> {
  const leaderboardStr = await context.redis.get('game:leaderboard');
  const leaderboard: LeaderboardEntry[] = leaderboardStr
    ? JSON.parse(leaderboardStr)
    : [];

  const index = leaderboard.findIndex(e => e.userId === userId);
  return index === -1 ? -1 : index + 1;
}
```

### Pattern 3: Matchmaking

```typescript
interface MatchmakingQueue {
  players: QueuedPlayer[];
}

interface QueuedPlayer {
  userId: string;
  username: string;
  queuedAt: number;
  skillLevel?: number;
}

async function joinMatchmaking(
  context: Context,
  userId: string,
  username: string
): Promise<{ matched: boolean; gameId?: string; opponent?: string }> {
  const queueKey = 'matchmaking:queue';
  const queueStr = await context.redis.get(queueKey);
  const queue: MatchmakingQueue = queueStr
    ? JSON.parse(queueStr)
    : { players: [] };

  // Check if already in queue
  if (queue.players.some(p => p.userId === userId)) {
    return { matched: false };
  }

  // Try to find opponent
  const availableOpponents = queue.players.filter(p => p.userId !== userId);

  if (availableOpponents.length > 0) {
    // Match with first available player
    const opponent = availableOpponents[0];

    // Remove opponent from queue
    queue.players = queue.players.filter(p => p.userId !== opponent.userId);
    await context.redis.set(queueKey, JSON.stringify(queue));

    // Create game
    const gameId = `game_${Date.now()}`;
    const game = await createGame(context, gameId);
    await joinGame(context, gameId, userId, username);
    await joinGame(context, gameId, opponent.userId, opponent.username);

    return {
      matched: true,
      gameId,
      opponent: opponent.username,
    };
  } else {
    // Add to queue
    queue.players.push({
      userId,
      username,
      queuedAt: Date.now(),
    });

    await context.redis.set(queueKey, JSON.stringify(queue));

    return { matched: false };
  }
}

async function leaveMatchmaking(context: Context, userId: string): Promise<void> {
  const queueKey = 'matchmaking:queue';
  const queueStr = await context.redis.get(queueKey);
  const queue: MatchmakingQueue = queueStr
    ? JSON.parse(queueStr)
    : { players: [] };

  queue.players = queue.players.filter(p => p.userId !== userId);

  await context.redis.set(queueKey, JSON.stringify(queue));
}
```

## Advanced State Management

### Pattern 1: Global State with Context

```typescript
// Create a state management service
interface AppState {
  theme: 'light' | 'dark';
  notifications: boolean;
  lastSync: number;
}

async function getAppState(context: Context, userId: string): Promise<AppState> {
  const key = `state:${userId}`;
  const stateStr = await context.redis.get(key);

  if (!stateStr) {
    return {
      theme: 'light',
      notifications: true,
      lastSync: Date.now(),
    };
  }

  return JSON.parse(stateStr);
}

async function updateAppState(
  context: Context,
  userId: string,
  updates: Partial<AppState>
): Promise<AppState> {
  const current = await getAppState(context, userId);
  const newState = { ...current, ...updates, lastSync: Date.now() };

  await context.redis.set(`state:${userId}`, JSON.stringify(newState));

  return newState;
}
```

### Pattern 2: Optimistic Updates

```typescript
// In custom post, update UI immediately, sync later
Devvit.addCustomPostType({
  name: 'optimistic-counter',
  height: 'regular',
  render: (context) => {
    const [localCount, setLocalCount] = context.useState(0);
    const [syncing, setSyncing] = context.useState(false);

    // Load initial count
    context.useAsync(async () => {
      const count = await context.redis.get('counter');
      setLocalCount(Number(count || 0));
    });

    const increment = async () => {
      // Optimistic update
      const newCount = localCount + 1;
      setLocalCount(newCount);
      setSyncing(true);

      try {
        // Sync to server
        const serverCount = await context.redis.incrBy('counter', 1);

        // If different from optimistic value, correct it
        if (serverCount !== newCount) {
          setLocalCount(serverCount);
        }
      } catch (error) {
        // Rollback on error
        setLocalCount(localCount);
        context.ui.showToast('Failed to sync');
      } finally {
        setSyncing(false);
      }
    };

    return (
      <vstack padding="medium" alignment="center middle" gap="medium">
        <text size="xxlarge">{localCount}</text>
        {syncing && <text size="small">Syncing...</text>}
        <button onPress={increment}>Increment</button>
      </vstack>
    );
  },
});
```

## Platform Limitations and Workarounds

Understanding what Devvit **cannot** do helps you design better apps.

### Limitation 1: No NPM Packages

**Problem:** Can't use most npm libraries.

**Workarounds:**
1. Use Devvit's built-in APIs
2. Implement functionality yourself (often simpler than you think)
3. Call external APIs that do the work
4. Use Web Views with full npm access

```typescript
// Instead of moment.js
const formatDate = (date: Date): string => {
  return date.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  });
};

// Instead of lodash
const debounce = <T extends (...args: any[]) => any>(
  func: T,
  wait: number
): ((...args: Parameters<T>) => void) => {
  let timeout: NodeJS.Timeout | null = null;

  return (...args: Parameters<T>) => {
    if (timeout) clearTimeout(timeout);
    timeout = setTimeout(() => func(...args), wait);
  };
};
```

### Limitation 2: No Persistent Connections

**Problem:** No WebSockets, no long-polling.

**Workarounds:**
1. Short polling with `useInterval`
2. Redis-based message passing
3. Scheduled jobs for async work

### Limitation 3: Execution Time Limits

**Problem:** Functions timeout after 3-5 seconds.

**Workarounds:**
1. Break work into chunks
2. Use scheduled jobs for heavy work
3. Process in batches

```typescript
// Bad: Try to process everything
async function processAllPosts(context: Context) {
  const posts = await getAllPosts(); // Could be thousands
  for (const post of posts) {
    await process(post); // Times out!
  }
}

// Good: Process in scheduled job with batches
Devvit.addSchedulerJob({
  name: 'processPosts',
  cron: '*/5 * * * *', // Every 5 minutes
  onRun: async (event, context) => {
    // Process only 50 at a time
    const offset = Number(await context.redis.get('process-offset') || '0');
    const batch = await getPostsBatch(offset, 50);

    for (const post of batch) {
      await process(post);
    }

    // Save progress
    await context.redis.set('process-offset', String(offset + 50));
  },
});
```

### Limitation 4: No File System

**Problem:** Can't read/write local files.

**Workarounds:**
1. Use Redis for storage
2. Use external APIs for file operations
3. Use media upload API for images

### Limitation 5: Limited HTTP Destinations

**Problem:** Some domains may be blocked.

**Workarounds:**
1. Use well-known public APIs
2. Proxy through your own server if needed
3. Check Reddit's allowed domains list

### Limitation 6: No Direct Database Access

**Problem:** Can't connect to PostgreSQL, MySQL, etc.

**Workarounds:**
1. Use Redis for most use cases
2. Call your own API that accesses the database
3. Design around Redis's capabilities

## Key Takeaways

1. **HTTP requests** - Use fetch() for external APIs
2. **App settings** - User-configurable via devvit.yaml
3. **Media handling** - Upload and use images
4. **Real-time patterns** - Polling, Redis pub/sub simulation, presence detection
5. **Game architecture** - Turn-based systems, leaderboards, matchmaking
6. **Advanced state** - Global state management, optimistic updates
7. **Platform limitations** - Understand constraints and workarounds
8. **Performance** - Parallelize operations, cache data
9. **Security** - Validate input, check permissions, protect secrets
10. **Error handling** - Comprehensive try-catch, retry logic

## Checkpoint Questions

1. How do you enable HTTP requests in your app?
2. Where do you define app settings?
3. What's the benefit of parallel requests?
4. How should you handle secrets like API keys?
5. What's a good project structure for larger apps?

<details>
<summary>Click to see answers</summary>

1. Add `http: true` to `Devvit.configure()`
2. In `devvit.yaml` under the `settings` section
3. Parallel requests with `Promise.all()` execute simultaneously, reducing total execution time
4. Store in app settings with `isSecret: true`, access via `context.settings.get()`, never log or store unencrypted
5. Separate folders for handlers, services, utils, types, and components

</details>

## Additional Resources

- [HTTP Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- [TypeScript Best Practices](https://www.typescriptlang.org/docs/handbook/declaration-files/do-s-and-don-ts.html)
- [Promise.all() Documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)

---

➡️ **Continue to [Lesson 10A: Testing and Quality Assurance](./10a-testing.md)**

**Estimated time to complete**: 3 hours
**Prerequisites**: Lessons 1-8 completed

**What you learned:**
- HTTP requests and external API integration
- App settings and configuration
- Real-time patterns (polling, pub/sub, presence)
- Game architecture (turn-based, leaderboards, matchmaking)
- Advanced state management
- Platform limitations and workarounds
- Security and performance best practices
