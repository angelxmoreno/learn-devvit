# Lesson 3: Your First Devvit App

**Duration**: 1 hour
**Level**: Beginner

## Overview

In this lesson, you'll build your first working Devvit app from scratch. You'll learn the basic structure, create a menu action, and see your code running in the Reddit UI.

## What We'll Build

A simple app that adds a "Say Hello" option to post menus. When clicked, it will display a toast notification with the post's title.

**Features:**
- Custom menu item on posts
- Access post information
- Display user feedback via toast

## Step 1: Create the Project

If you haven't already from Lesson 2, create a new project:

```bash
devvit new hello-devvit
cd hello-devvit
npm install
```

When prompted:
- **Template**: Select "Empty"
- **Name**: hello-devvit
- **Description**: My first Devvit application

## Step 2: Understanding the Entry Point

Open `src/main.tsx` in your editor. You'll see:

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
});

export default Devvit;
```

Let's break this down:

### Import Statement
```typescript
import { Devvit } from '@devvit/public-api';
```
- Imports the main Devvit object
- All functionality comes from this package
- TypeScript provides full type safety

### Configuration
```typescript
Devvit.configure({
  redditAPI: true,
});
```
- Enables access to Reddit's API
- Other options include: `redis`, `http`, `media`
- Only enable what you need for better performance

### Export
```typescript
export default Devvit;
```
- Required at the end of your main file
- Tells Devvit what to load

## Step 3: Add Your First Menu Action

A **menu action** adds a custom item to Reddit's context menus. Let's add one:

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
});

// Add a menu item to posts
Devvit.addMenuItem({
  label: 'Say Hello',
  location: 'post',
  onPress: async (event, context) => {
    // Get the post that was clicked
    const post = await context.reddit.getPostById(event.targetId);

    // Show a toast notification
    context.ui.showToast(`Hello! This post is titled: "${post.title}"`);
  },
});

export default Devvit;
```

### Anatomy of `addMenuItem`

**`label`** - The text shown in the menu
```typescript
label: 'Say Hello'
```

**`location`** - Where the menu appears
- `'post'` - Post context menu
- `'comment'` - Comment context menu
- `'subreddit'` - Subreddit menu

**`onPress`** - What happens when clicked
```typescript
onPress: async (event, context) => {
  // Your code here
}
```

### The Event Parameter

Contains information about what triggered the action:

```typescript
event.targetId      // ID of the post/comment clicked
event.location      // Where it was triggered
event.userDisplayName  // Who clicked it
```

### The Context Parameter

Your interface to Reddit and Devvit features:

```typescript
context.reddit      // Reddit API access
context.ui          // UI operations (toasts, forms)
context.redis       // Redis storage
context.scheduler   // Job scheduling
context.settings    // App settings
```

## Step 4: Test Locally

Start the development server:

```bash
npm run dev
```

or

```bash
devvit dev
```

Output should look like:
```
✓ Building app...
✓ App built successfully
✓ Starting development server...
✓ Server running at http://localhost:3000
```

### Using the Playground

1. Open your browser to `http://localhost:3000`
2. You'll see a simulated Reddit post
3. Click the "..." menu on the post
4. You should see your "Say Hello" option
5. Click it to see the toast notification

**Hot Reload**: Edit `main.tsx` and save - changes appear automatically!

## Step 5: Improve the App

Let's add more functionality. Update your code:

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
});

Devvit.addMenuItem({
  label: 'Post Info',
  location: 'post',
  forUserType: 'moderator',  // Only show to moderators
  onPress: async (event, context) => {
    const post = await context.reddit.getPostById(event.targetId);

    // Calculate post age
    const ageMs = Date.now() - post.createdAt.getTime();
    const ageHours = Math.floor(ageMs / (1000 * 60 * 60));

    // Build info message
    const info = [
      `**${post.title}**`,
      ``,
      `👤 Author: u/${post.authorName}`,
      `⬆️ Score: ${post.score}`,
      `💬 Comments: ${post.numberOfComments}`,
      `🕐 Age: ${ageHours} hours`,
      `🏷️ Flair: ${post.flair?.text || 'None'}`,
    ].join('\n');

    context.ui.showToast(info);
  },
});

export default Devvit;
```

### New Concepts Here:

**`forUserType`** - Restricts who sees the action
- `'moderator'` - Only subreddit mods
- `'member'` - Any subreddit member
- Leave empty for everyone

**Post Properties** - TypeScript knows all available fields:
```typescript
post.title           // string
post.authorName      // string
post.score           // number
post.numberOfComments // number
post.createdAt       // Date
post.flair           // { text: string } | undefined
post.url             // string
post.permalink       // string
```

**Multi-line Toast** - Use `\n` for line breaks

## Step 6: Add Error Handling

Always handle potential errors in production apps:

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
});

Devvit.addMenuItem({
  label: 'Post Info',
  location: 'post',
  forUserType: 'moderator',
  onPress: async (event, context) => {
    try {
      const post = await context.reddit.getPostById(event.targetId);

      // Check if post exists
      if (!post) {
        context.ui.showToast('⚠️ Could not find post');
        return;
      }

      const ageMs = Date.now() - post.createdAt.getTime();
      const ageHours = Math.floor(ageMs / (1000 * 60 * 60));

      const info = [
        `**${post.title}**`,
        ``,
        `👤 Author: u/${post.authorName}`,
        `⬆️ Score: ${post.score}`,
        `💬 Comments: ${post.numberOfComments}`,
        `🕐 Age: ${ageHours} hours`,
        `🏷️ Flair: ${post.flair?.text || 'None'}`,
      ].join('\n');

      context.ui.showToast({
        text: info,
        appearance: 'success',
      });

    } catch (error) {
      console.error('Error fetching post:', error);
      context.ui.showToast({
        text: '❌ An error occurred',
        appearance: 'error',
      });
    }
  },
});

export default Devvit;
```

### Toast Appearance Options

```typescript
context.ui.showToast({
  text: 'Message',
  appearance: 'success'  // 'success', 'error', 'neutral'
});
```

## Step 7: Add Multiple Actions

You can add as many menu items as you want:

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
});

// Action 1: Post Info
Devvit.addMenuItem({
  label: 'Post Info',
  location: 'post',
  onPress: async (event, context) => {
    const post = await context.reddit.getPostById(event.targetId);
    context.ui.showToast(`Score: ${post.score}, Comments: ${post.numberOfComments}`);
  },
});

// Action 2: Author Info
Devvit.addMenuItem({
  label: 'Author Info',
  location: 'post',
  onPress: async (event, context) => {
    const post = await context.reddit.getPostById(event.targetId);
    context.ui.showToast(`Posted by u/${post.authorName}`);
  },
});

// Action 3: Comment Menu
Devvit.addMenuItem({
  label: 'Comment Details',
  location: 'comment',
  onPress: async (event, context) => {
    const comment = await context.reddit.getCommentById(event.targetId);
    context.ui.showToast(`Comment by u/${comment.authorName}: ${comment.body.slice(0, 50)}...`);
  },
});

export default Devvit;
```

## Step 8: Type Safety with TypeScript

Devvit provides excellent TypeScript support. Let's see it in action:

```typescript
import { Devvit, Context, MenuItemOnPressEvent } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
});

// Extract handler to separate function for reusability
async function handlePostInfo(
  event: MenuItemOnPressEvent,
  context: Context
): Promise<void> {
  const post = await context.reddit.getPostById(event.targetId);

  // TypeScript knows all properties of 'post'
  const info: string = `${post.title} - ${post.score} points`;

  context.ui.showToast(info);
}

Devvit.addMenuItem({
  label: 'Post Info',
  location: 'post',
  onPress: handlePostInfo,
});

export default Devvit;
```

**Benefits:**
- Autocomplete in your IDE
- Compile-time error checking
- Refactoring support
- Better documentation

## Step 9: Project Organization

For larger apps, organize your code into modules:

```
src/
├── main.tsx              # Entry point
├── handlers/
│   ├── postHandlers.ts   # Post-related actions
│   └── commentHandlers.ts # Comment-related actions
└── utils/
    └── formatting.ts     # Shared utilities
```

**Example: `src/utils/formatting.ts`**
```typescript
export function formatPostAge(createdAt: Date): string {
  const ageMs = Date.now() - createdAt.getTime();
  const ageHours = Math.floor(ageMs / (1000 * 60 * 60));

  if (ageHours < 24) {
    return `${ageHours} hours ago`;
  }

  const ageDays = Math.floor(ageHours / 24);
  return `${ageDays} days ago`;
}

export function formatNumber(num: number): string {
  if (num >= 1000000) {
    return `${(num / 1000000).toFixed(1)}M`;
  }
  if (num >= 1000) {
    return `${(num / 1000).toFixed(1)}K`;
  }
  return num.toString();
}
```

**Example: `src/handlers/postHandlers.ts`**
```typescript
import { Context, MenuItemOnPressEvent } from '@devvit/public-api';
import { formatPostAge, formatNumber } from '../utils/formatting.js';

export async function handlePostInfo(
  event: MenuItemOnPressEvent,
  context: Context
): Promise<void> {
  const post = await context.reddit.getPostById(event.targetId);

  const info = [
    `**${post.title}**`,
    ``,
    `👤 u/${post.authorName}`,
    `⬆️ ${formatNumber(post.score)} points`,
    `💬 ${formatNumber(post.numberOfComments)} comments`,
    `🕐 ${formatPostAge(post.createdAt)}`,
  ].join('\n');

  context.ui.showToast(info);
}
```

**Updated `src/main.tsx`**
```typescript
import { Devvit } from '@devvit/public-api';
import { handlePostInfo } from './handlers/postHandlers.js';

Devvit.configure({
  redditAPI: true,
});

Devvit.addMenuItem({
  label: 'Post Info',
  location: 'post',
  onPress: handlePostInfo,
});

export default Devvit;
```

**Important**: Use `.js` extensions in imports even though you're writing `.ts` files. TypeScript will handle the resolution.

## Step 10: Debugging Tips

### Console Logging

Use `console.log()` for debugging:

```typescript
Devvit.addMenuItem({
  label: 'Debug Post',
  location: 'post',
  onPress: async (event, context) => {
    console.log('Event:', event);
    console.log('Target ID:', event.targetId);

    const post = await context.reddit.getPostById(event.targetId);
    console.log('Post data:', post);

    context.ui.showToast('Check the console!');
  },
});
```

Logs appear in:
- **Local dev**: Your terminal running `devvit dev`
- **Production**: Use `devvit logs <app-name>`

### Type Checking

Run type checking without building:

```bash
npx tsc --noEmit
```

Or add to `package.json`:
```json
{
  "scripts": {
    "typecheck": "tsc --noEmit"
  }
}
```

Then run:
```bash
npm run typecheck
```

## Common Patterns

### Pattern 1: Check User Permissions

```typescript
Devvit.addMenuItem({
  label: 'Mod Action',
  location: 'post',
  onPress: async (event, context) => {
    const subreddit = await context.reddit.getCurrentSubreddit();
    const user = await context.reddit.getCurrentUser();
    const isMod = await user.isModerator(subreddit.name);

    if (!isMod) {
      context.ui.showToast('⚠️ Moderators only');
      return;
    }

    // Mod-only logic here
  },
});
```

### Pattern 2: Batch Operations

```typescript
Devvit.addMenuItem({
  label: 'Get Top Comments',
  location: 'post',
  onPress: async (event, context) => {
    const post = await context.reddit.getPostById(event.targetId);
    const comments = await post.comments.all();

    const topComments = comments
      .sort((a, b) => b.score - a.score)
      .slice(0, 5);

    const summary = topComments
      .map((c, i) => `${i + 1}. ${c.score} pts - ${c.body.slice(0, 50)}...`)
      .join('\n');

    context.ui.showToast(summary);
  },
});
```

### Pattern 3: Conditional Logic

```typescript
Devvit.addMenuItem({
  label: 'Analyze Post',
  location: 'post',
  onPress: async (event, context) => {
    const post = await context.reddit.getPostById(event.targetId);

    let message = '';

    if (post.score > 1000) {
      message = '🔥 Hot post!';
    } else if (post.score < 0) {
      message = '📉 Controversial post';
    } else {
      message = '📊 Normal post';
    }

    if (post.locked) {
      message += ' (Locked)';
    }

    if (post.removed) {
      message += ' (Removed)';
    }

    context.ui.showToast(message);
  },
});
```

## Practice Exercises

Try building these yourself:

### Exercise 1: Comment Counter
Add a menu action that counts how many comments a post has and shows the percentage that are from the OP.

<details>
<summary>Solution</summary>

```typescript
Devvit.addMenuItem({
  label: 'Count OP Comments',
  location: 'post',
  onPress: async (event, context) => {
    const post = await context.reddit.getPostById(event.targetId);
    const comments = await post.comments.all();

    const opComments = comments.filter(c => c.authorName === post.authorName);
    const percentage = ((opComments.length / comments.length) * 100).toFixed(1);

    context.ui.showToast(
      `${comments.length} total comments\n` +
      `${opComments.length} from OP (${percentage}%)`
    );
  },
});
```
</details>

### Exercise 2: Post Age Alert
Create a menu action that alerts if a post is older than 24 hours.

<details>
<summary>Solution</summary>

```typescript
Devvit.addMenuItem({
  label: 'Check Age',
  location: 'post',
  onPress: async (event, context) => {
    const post = await context.reddit.getPostById(event.targetId);
    const ageMs = Date.now() - post.createdAt.getTime();
    const ageHours = ageMs / (1000 * 60 * 60);

    if (ageHours > 24) {
      context.ui.showToast({
        text: `⚠️ This post is ${Math.floor(ageHours / 24)} days old`,
        appearance: 'error',
      });
    } else {
      context.ui.showToast({
        text: `✓ Post is ${Math.floor(ageHours)} hours old`,
        appearance: 'success',
      });
    }
  },
});
```
</details>

### Exercise 3: Flair Checker
Add a menu action that shows what percentage of a post's comments come from users with flair.

## Key Takeaways

1. **Entry point** - `main.tsx` configures and exports your app
2. **Menu actions** - Easy way to add functionality to Reddit UI
3. **Event & Context** - Your interface to Reddit data and features
4. **TypeScript** - Full type safety for better development
5. **Error handling** - Always wrap API calls in try-catch
6. **Organization** - Split large apps into modules
7. **Testing** - Use `devvit dev` for rapid local development

## Checkpoint Questions

1. What method adds a menu item to posts?
2. What are the two parameters in an `onPress` handler?
3. How do you access post data in a menu action?
4. What command starts the local development server?
5. How do you show a toast notification?

<details>
<summary>Click to see answers</summary>

1. `Devvit.addMenuItem()`
2. `event` and `context`
3. `await context.reddit.getPostById(event.targetId)`
4. `devvit dev` or `npm run dev`
5. `context.ui.showToast('message')` or `context.ui.showToast({ text: 'message', appearance: 'success' })`

</details>

## Additional Resources

- [Menu Items Documentation](https://developers.reddit.com/docs/menu-actions)
- [Reddit API Reference](https://developers.reddit.com/docs/reddit-api)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/)

## Next Steps

You've built your first working Devvit app! Next, we'll dive deeper into Devvit's architecture and understand how everything works under the hood.

---

➡️ **Continue to [Lesson 4: Understanding Devvit Architecture](./04-architecture.md)**

**Estimated time to complete**: 1 hour
**Practice exercises**: 3 hands-on challenges
**Prerequisites**: Lessons 1-2 completed
