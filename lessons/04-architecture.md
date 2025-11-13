# Lesson 4: Understanding Devvit Architecture

**Duration**: 1.5 hours
**Level**: Intermediate

## Overview

In this lesson, you'll learn how Devvit apps work under the hood. Understanding the architecture will help you build more efficient, reliable, and powerful applications.

## The Devvit Execution Model

### Serverless Architecture

Devvit apps run on Reddit's infrastructure using a **serverless, event-driven** model:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Trigger   │ --> │  Your Code  │ --> │   Action    │
│  (Event)    │     │  (Handler)  │     │  (Result)   │
└─────────────┘     └─────────────┘     └─────────────┘
```

**Key characteristics:**

1. **Stateless** - Each execution starts fresh, no memory between runs
2. **Isolated** - Your code runs in a sandbox with limited access
3. **Time-limited** - Must complete within timeout (typically 3-5 seconds)
4. **Event-driven** - Triggered by specific Reddit events
5. **Scalable** - Reddit handles all scaling automatically

### Execution Lifecycle

```typescript
// 1. Event occurs (user clicks menu item)
User clicks "Say Hello"
    ↓
// 2. Devvit initializes your app
App code loads, Devvit.configure() runs
    ↓
// 3. Handler executes
onPress() function runs
    ↓
// 4. API calls made
context.reddit.getPostById()
    ↓
// 5. Response returned
context.ui.showToast()
    ↓
// 6. Environment terminated
Memory cleared, no state persists
```

**Important**: State does NOT persist between executions!

## Triggers: How Apps Start

Triggers are events that cause your code to execute. Devvit supports several trigger types:

### 1. Menu Items (Actions)

Add custom items to Reddit menus:

```typescript
Devvit.addMenuItem({
  label: 'My Action',
  location: 'post',  // or 'comment', 'subreddit'
  onPress: async (event, context) => {
    // Runs when user clicks the menu item
  },
});
```

**Locations:**
- `post` - Post overflow menu (...)
- `comment` - Comment overflow menu
- `subreddit` - Subreddit menu (sidebar)

### 2. Post Submit

Trigger when users create posts:

```typescript
Devvit.addTrigger({
  event: 'PostSubmit',
  onEvent: async (event, context) => {
    const post = event.post;
    console.log(`New post created: ${post.title}`);

    // Auto-flair, validate, moderate, etc.
  },
});
```

**Use cases:**
- Auto-flair posts
- Content validation
- Welcome messages
- Statistics tracking

### 3. Comment Submit

Trigger when users comment:

```typescript
Devvit.addTrigger({
  event: 'CommentSubmit',
  onEvent: async (event, context) => {
    const comment = event.comment;

    // Auto-reply, analyze sentiment, etc.
  },
});
```

### 4. Post Update

Trigger when posts are edited:

```typescript
Devvit.addTrigger({
  event: 'PostUpdate',
  onEvent: async (event, context) => {
    const post = event.post;
    // Handle edits, update flair, etc.
  },
});
```

### 5. Comment Update

Trigger when comments are edited:

```typescript
Devvit.addTrigger({
  event: 'CommentUpdate',
  onEvent: async (event, context) => {
    const comment = event.comment;
    // Track edits, check for rule violations
  },
});
```

### 6. Mod Actions

Trigger on moderation actions:

```typescript
Devvit.addTrigger({
  event: 'ModAction',
  onEvent: async (event, context) => {
    console.log(`Mod action: ${event.action}`);
    // Log actions, send notifications, etc.
  },
});
```

**Action types:**
- `approvelink`, `removelink` - Post approval/removal
- `approvecomment`, `removecomment` - Comment approval/removal
- `banuser`, `unbanuser` - User bans
- `lock`, `unlock` - Lock/unlock posts
- `distinguish`, `undistinguish` - Mod distinguish
- And many more...

### 7. Scheduled Jobs

Trigger at specific times (cron-style):

```typescript
Devvit.addSchedulerJob({
  name: 'dailyTask',
  cron: '0 9 * * *',  // Every day at 9am
  onRun: async (event, context) => {
    // Create daily thread, generate reports, etc.
  },
});
```

We'll cover scheduling in detail in Lesson 8.

### 8. App Install/Upgrade

Trigger when app is installed or upgraded:

```typescript
Devvit.addTrigger({
  event: 'AppInstall',
  onEvent: async (event, context) => {
    // Initialize data, create welcome post, etc.
  },
});

Devvit.addTrigger({
  event: 'AppUpgrade',
  onEvent: async (event, context) => {
    // Migrate data, update configurations
  },
});
```

## The Context Object

The `context` parameter is your gateway to Devvit's capabilities:

```typescript
interface Context {
  reddit: RedditAPIClient;
  ui: UIClient;
  redis: RedisClient;
  scheduler: SchedulerClient;
  settings: SettingsClient;
  assets: AssetsClient;
  dimensions?: Dimensions;
}
```

### context.reddit - Reddit API

Access all Reddit functionality:

```typescript
// Get objects
const post = await context.reddit.getPostById('t3_abc123');
const comment = await context.reddit.getCommentById('t1_xyz789');
const user = await context.reddit.getUserById('t2_user123');
const subreddit = await context.reddit.getSubredditById('t5_sub456');

// Get current context
const currentSub = await context.reddit.getCurrentSubreddit();
const currentUser = await context.reddit.getCurrentUser();

// Create content
const newPost = await context.reddit.submitPost({
  title: 'Hello World',
  subredditName: 'test',
  text: 'This is my post',
});

const newComment = await context.reddit.submitComment({
  id: post.id,
  text: 'Great post!',
});

// Moderation actions
await post.remove();
await post.approve();
await post.lock();
await comment.remove();
await user.ban({ subredditName: 'test', reason: 'Spam' });
```

### context.ui - User Interface

Show UI elements to users:

```typescript
// Toast notifications
context.ui.showToast('Simple message');
context.ui.showToast({
  text: 'Styled message',
  appearance: 'success',  // 'success', 'error', 'neutral'
});

// Show forms (covered in Lesson 5)
context.ui.showForm(myForm);

// Navigate (in custom posts)
context.ui.navigateTo('https://reddit.com/r/devvit');
```

### context.redis - Data Storage

Persistent key-value storage:

```typescript
// Set values
await context.redis.set('key', 'value');
await context.redis.set('counter', '0');

// Get values
const value = await context.redis.get('key');
const counter = await context.redis.get('counter');

// Increment
await context.redis.incrBy('counter', 1);

// Delete
await context.redis.del('key');

// Check existence
const exists = await context.redis.exists('key');
```

We'll cover Redis in depth in Lesson 7.

### context.scheduler - Job Scheduling

Schedule future tasks:

```typescript
// Schedule a one-time job
await context.scheduler.runJob({
  name: 'reminderJob',
  data: { userId: 'abc123' },
  runAt: new Date(Date.now() + 3600000),  // 1 hour from now
});

// Cancel a job
await context.scheduler.cancelJob('job-id');
```

More in Lesson 8.

### context.settings - App Configuration

Access user-configurable settings:

```typescript
// Get a setting value
const apiKey = await context.settings.get('apiKey');
const threshold = await context.settings.get('threshold');
```

Settings are configured in `devvit.yaml`:

```yaml
settings:
  - name: apiKey
    type: string
    label: API Key
    description: Your external API key
  - name: threshold
    type: number
    label: Score Threshold
    defaultValue: 100
```

## Configuration Options

The `Devvit.configure()` call enables features:

```typescript
Devvit.configure({
  redditAPI: true,      // Access to Reddit API
  redis: true,          // Redis storage
  http: true,           // HTTP fetch
  media: true,          // Media upload/processing
});
```

**Only enable what you need** - It improves performance and reduces permissions.

### Examples:

**Simple menu action** (no storage):
```typescript
Devvit.configure({
  redditAPI: true,
});
```

**Bot with storage**:
```typescript
Devvit.configure({
  redditAPI: true,
  redis: true,
});
```

**External API integration**:
```typescript
Devvit.configure({
  redditAPI: true,
  http: true,
});
```

**Media handler**:
```typescript
Devvit.configure({
  redditAPI: true,
  media: true,
});
```

## Event Objects

Different triggers provide different event data:

### MenuItemEvent

```typescript
interface MenuItemOnPressEvent {
  targetId: string;           // ID of the thing clicked (post/comment/subreddit)
  location: string;           // Where action triggered
  userDisplayName?: string;   // Who clicked
}
```

### PostSubmitEvent

```typescript
interface PostSubmitEvent {
  post: Post;                 // The submitted post
  author: User;               // Post author
  subreddit: Subreddit;       // Where it was posted
}
```

### CommentSubmitEvent

```typescript
interface CommentSubmitEvent {
  comment: Comment;           // The submitted comment
  post: Post;                 // Parent post
  author: User;               // Comment author
  subreddit: Subreddit;       // Where it was posted
}
```

### ModActionEvent

```typescript
interface ModActionEvent {
  action: string;             // Type of action (e.g., 'removelink')
  moderator: User;            // Who performed the action
  target?: Post | Comment;    // What was acted upon
  subreddit: Subreddit;       // Where action occurred
}
```

## Async Patterns

Devvit is heavily async. Understanding promises is crucial:

### Basic Async/Await

```typescript
Devvit.addMenuItem({
  label: 'Fetch Data',
  location: 'post',
  onPress: async (event, context) => {
    // Await single operation
    const post = await context.reddit.getPostById(event.targetId);

    context.ui.showToast(post.title);
  },
});
```

### Sequential Operations

When one operation depends on another:

```typescript
async function processPost(postId: string, context: Context) {
  // Must wait for post before getting author
  const post = await context.reddit.getPostById(postId);
  const author = await context.reddit.getUserById(post.authorId);

  // Must wait for storage before incrementing
  const count = await context.redis.get('post-count');
  await context.redis.set('post-count', String(Number(count || 0) + 1));

  return { post, author };
}
```

### Parallel Operations

When operations are independent, run them in parallel:

```typescript
async function fetchMultiple(context: Context) {
  // Bad: Sequential (slow)
  const post1 = await context.reddit.getPostById('id1');
  const post2 = await context.reddit.getPostById('id2');
  const post3 = await context.reddit.getPostById('id3');

  // Good: Parallel (fast)
  const [post1, post2, post3] = await Promise.all([
    context.reddit.getPostById('id1'),
    context.reddit.getPostById('id2'),
    context.reddit.getPostById('id3'),
  ]);

  return [post1, post2, post3];
}
```

### Error Handling with Async

Always handle potential failures:

```typescript
Devvit.addMenuItem({
  label: 'Safe Fetch',
  location: 'post',
  onPress: async (event, context) => {
    try {
      const post = await context.reddit.getPostById(event.targetId);
      const comments = await post.comments.all();

      context.ui.showToast(`${comments.length} comments`);
    } catch (error) {
      console.error('Error:', error);
      context.ui.showToast({
        text: 'Failed to fetch data',
        appearance: 'error',
      });
    }
  },
});
```

## Execution Constraints

Understanding limitations helps you build reliable apps:

### Time Limits

- **Typical timeout**: 3-5 seconds
- **Strategy**: Keep operations quick
- **For long tasks**: Use scheduled jobs

```typescript
// Bad: Too slow
Devvit.addMenuItem({
  label: 'Process All Posts',
  location: 'subreddit',
  onPress: async (event, context) => {
    const sub = await context.reddit.getCurrentSubreddit();
    const posts = await sub.getHotPosts().all();  // Could be thousands!

    // Will likely timeout
    for (const post of posts) {
      await processPost(post);
    }
  },
});

// Good: Process in chunks with scheduled jobs
Devvit.addMenuItem({
  label: 'Start Processing',
  location: 'subreddit',
  onPress: async (event, context) => {
    // Schedule background job to process
    await context.scheduler.runJob({
      name: 'processChunk',
      data: { offset: 0 },
      runAt: new Date(),
    });

    context.ui.showToast('Processing started!');
  },
});
```

### Memory Limits

- **Stateless**: No memory between executions
- **Solution**: Use Redis for persistence

```typescript
// Bad: State doesn't persist
let counter = 0;  // Resets to 0 every execution!

Devvit.addMenuItem({
  label: 'Count',
  location: 'post',
  onPress: async (event, context) => {
    counter++;  // Always shows 1
    context.ui.showToast(`Count: ${counter}`);
  },
});

// Good: Use Redis
Devvit.addMenuItem({
  label: 'Count',
  location: 'post',
  onPress: async (event, context) => {
    const count = await context.redis.incrBy('counter', 1);
    context.ui.showToast(`Count: ${count}`);
  },
});
```

### No File System

- **Can't**: Read/write local files
- **Can**: Use Redis for data, HTTP for external APIs

```typescript
// Bad: File system not available
import fs from 'fs';  // Won't work!

// Good: Use Redis
await context.redis.set('data', JSON.stringify({ items: [] }));
const data = await context.redis.get('data');
const parsed = JSON.parse(data || '{}');
```

## App Configuration (devvit.yaml)

The `devvit.yaml` file configures your app:

```yaml
# Basic metadata
name: my-awesome-app
version: 1.0.0
author: your-username
description: Does amazing things

# User-configurable settings
settings:
  - name: welcomeMessage
    type: string
    label: Welcome Message
    description: Message shown to new users
    defaultValue: "Welcome!"

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

  - name: subredditList
    type: string
    label: Subreddit List
    description: Comma-separated subreddits

# Permissions (set via Devvit.configure)
permissions:
  - reddit-api
  - redis
```

## Best Practices

### 1. Keep Handlers Focused

```typescript
// Bad: Monolithic handler
Devvit.addMenuItem({
  label: 'Do Everything',
  location: 'post',
  onPress: async (event, context) => {
    // 200 lines of code...
  },
});

// Good: Modular
async function analyzePost(post: Post) { /* ... */ }
async function updateStorage(context: Context, data: any) { /* ... */ }
async function notifyUser(context: Context, message: string) { /* ... */ }

Devvit.addMenuItem({
  label: 'Analyze',
  location: 'post',
  onPress: async (event, context) => {
    const post = await context.reddit.getPostById(event.targetId);
    const analysis = await analyzePost(post);
    await updateStorage(context, analysis);
    await notifyUser(context, 'Analysis complete!');
  },
});
```

### 2. Handle Errors Gracefully

```typescript
async function safeHandler(event: MenuItemOnPressEvent, context: Context) {
  try {
    const post = await context.reddit.getPostById(event.targetId);

    if (!post) {
      context.ui.showToast('Post not found');
      return;
    }

    // Main logic...

  } catch (error) {
    console.error('Handler error:', error);
    context.ui.showToast({
      text: 'Something went wrong',
      appearance: 'error',
    });
  }
}
```

### 3. Use TypeScript Types

```typescript
import { Devvit, Context, Post } from '@devvit/public-api';

interface PostAnalysis {
  score: number;
  commentCount: number;
  age: number;
}

async function analyzePost(post: Post): Promise<PostAnalysis> {
  return {
    score: post.score,
    commentCount: post.numberOfComments,
    age: Date.now() - post.createdAt.getTime(),
  };
}
```

### 4. Optimize API Calls

```typescript
// Bad: Multiple calls
async function getPostDetails(postId: string, context: Context) {
  const post = await context.reddit.getPostById(postId);
  const author = await context.reddit.getUserById(post.authorId);
  const sub = await context.reddit.getSubredditById(post.subredditId);
  return { post, author, sub };
}

// Good: Parallel calls
async function getPostDetails(postId: string, context: Context) {
  const post = await context.reddit.getPostById(postId);

  const [author, sub] = await Promise.all([
    context.reddit.getUserById(post.authorId),
    context.reddit.getSubredditById(post.subredditId),
  ]);

  return { post, author, sub };
}
```

## Debugging Architecture Issues

### Issue: State Doesn't Persist

**Problem:**
```typescript
let myData = [];  // Resets every execution
```

**Solution:**
```typescript
// Store in Redis
await context.redis.set('myData', JSON.stringify([]));
```

### Issue: Timeout Errors

**Problem:** Too much work in one execution

**Solution:** Break into smaller jobs
```typescript
// Schedule background job instead
await context.scheduler.runJob({
  name: 'heavyWork',
  data: { task: 'process' },
  runAt: new Date(),
});
```

### Issue: NPM Package Not Found

**Problem:** Devvit doesn't support most npm packages

**Solution:** Use built-in APIs or approved packages

## Practice Exercises

### Exercise 1: Multi-Trigger App

Create an app that:
- Welcomes new posts with a comment (PostSubmit)
- Logs mod actions (ModAction)
- Has a menu item to show stats

<details>
<summary>Solution</summary>

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
  redis: true,
});

Devvit.addTrigger({
  event: 'PostSubmit',
  onEvent: async (event, context) => {
    await context.reddit.submitComment({
      id: event.post.id,
      text: 'Welcome! Thanks for posting.',
    });

    await context.redis.incrBy('totalPosts', 1);
  },
});

Devvit.addTrigger({
  event: 'ModAction',
  onEvent: async (event, context) => {
    await context.redis.incrBy('totalModActions', 1);
    console.log(`Mod action: ${event.action} by ${event.moderator.username}`);
  },
});

Devvit.addMenuItem({
  label: 'Show Stats',
  location: 'subreddit',
  onPress: async (event, context) => {
    const posts = await context.redis.get('totalPosts') || '0';
    const actions = await context.redis.get('totalModActions') || '0';

    context.ui.showToast(`Posts: ${posts}, Mod Actions: ${actions}`);
  },
});

export default Devvit;
```
</details>

### Exercise 2: Parallel Data Fetch

Fetch a post's top 3 comments and author info in parallel, then display.

<details>
<summary>Solution</summary>

```typescript
Devvit.addMenuItem({
  label: 'Quick Summary',
  location: 'post',
  onPress: async (event, context) => {
    const post = await context.reddit.getPostById(event.targetId);

    const [comments, author] = await Promise.all([
      post.comments.all(),
      context.reddit.getUserById(post.authorId),
    ]);

    const top3 = comments
      .sort((a, b) => b.score - a.score)
      .slice(0, 3);

    const summary = [
      `Post by u/${author.username}`,
      `Top comments:`,
      ...top3.map(c => `- ${c.score} pts: ${c.body.slice(0, 30)}...`),
    ].join('\n');

    context.ui.showToast(summary);
  },
});
```
</details>

## Key Takeaways

1. **Serverless & stateless** - No memory between executions
2. **Event-driven** - Code runs in response to triggers
3. **Time-limited** - Must complete quickly
4. **Context object** - Your interface to all Devvit features
5. **Async patterns** - Use Promise.all() for parallel operations
6. **Redis for persistence** - Store data between executions
7. **Error handling** - Always wrap in try-catch
8. **Configuration** - Enable only needed features

## Checkpoint Questions

1. What happens to variables between function executions?
2. Name three types of triggers available in Devvit.
3. What's the difference between sequential and parallel async operations?
4. How do you persist data between executions?
5. What's included in the `context` parameter?

<details>
<summary>Click to see answers</summary>

1. Variables are reset; memory doesn't persist between executions
2. Any three of: MenuItems, PostSubmit, CommentSubmit, ModAction, Scheduled Jobs, AppInstall
3. Sequential runs operations one after another (slower), parallel runs multiple operations simultaneously (faster)
4. Use `context.redis` for persistent key-value storage
5. reddit, ui, redis, scheduler, settings, assets (and optionally dimensions for custom posts)

</details>

## Additional Resources

- [Triggers Documentation](https://developers.reddit.com/docs/triggers)
- [Context API Reference](https://developers.reddit.com/docs/api/context)
- [TypeScript Async Patterns](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-1-7.html)

---

➡️ **Continue to [Lesson 5: Working with Forms and UI](./05-forms-and-ui.md)**

**Estimated time to complete**: 1.5 hours
**Practice exercises**: 2 hands-on challenges
**Prerequisites**: Lessons 1-3 completed
