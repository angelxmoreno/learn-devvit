# Lesson 9: Advanced Features and Best Practices

**Duration**: 2 hours
**Level**: Advanced

## Overview

Master advanced Devvit features including HTTP requests, media handling, app settings, and production-ready best practices.

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

## Key Takeaways

1. **HTTP requests** - Use fetch() for external APIs
2. **App settings** - User-configurable via devvit.yaml
3. **Media handling** - Upload and use images
4. **Performance** - Parallelize operations, cache data
5. **Security** - Validate input, check permissions, protect secrets
6. **Error handling** - Comprehensive try-catch, retry logic
7. **Code organization** - Modular structure, type safety
8. **Testing** - Debug logging, mock data

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

➡️ **Continue to [Lesson 10: Testing, Debugging, and Deployment](./10-deployment.md)**

**Estimated time to complete**: 2 hours
**Prerequisites**: Lessons 1-8 completed
