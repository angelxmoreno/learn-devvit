# Lesson 6: Reddit API Integration

**Duration**: 2 hours
**Level**: Intermediate

## Overview

In this lesson, you'll master the Reddit API through Devvit. You'll learn how to work with posts, comments, users, subreddits, and moderation actions.

## The Reddit API Client

Access the Reddit API through `context.reddit`:

```typescript
Devvit.addMenuItem({
  label: 'Use Reddit API',
  location: 'post',
  onPress: async (event, context) => {
    // context.reddit is your gateway to Reddit
    const post = await context.reddit.getPostById(event.targetId);
  },
});
```

## Working with Posts

### Getting Posts

**Get post by ID:**
```typescript
const post = await context.reddit.getPostById('t3_abc123');

// Access post properties
console.log(post.title);
console.log(post.score);
console.log(post.authorName);
console.log(post.url);
console.log(post.permalink);
```

**Get current post (in custom post context):**
```typescript
const post = await context.reddit.getPostById(context.postId!);
```

**Get posts from subreddit:**
```typescript
const subreddit = await context.reddit.getSubredditById('t5_2qh33');

// Hot posts
const hotPosts = await subreddit.getHotPosts({
  limit: 10,
  pageSize: 10,
}).all();

// New posts
const newPosts = await subreddit.getNewPosts({
  limit: 10,
}).all();

// Top posts
const topPosts = await subreddit.getTopPosts({
  timeframe: 'day',  // 'hour', 'day', 'week', 'month', 'year', 'all'
  limit: 10,
}).all();

// Rising posts
const risingPosts = await subreddit.getRisingPosts({
  limit: 10,
}).all();

// Controversial posts
const controversialPosts = await subreddit.getControversialPosts({
  timeframe: 'day',
  limit: 10,
}).all();
```

### Post Properties

```typescript
interface Post {
  id: string;                    // Fullname (e.g., 't3_abc123')
  title: string;                 // Post title
  body?: string;                 // Self-post text (if text post)
  url: string;                   // Link URL (if link post)
  authorName: string;            // Username of author
  authorId: string;              // Author's user ID
  subredditName: string;         // Subreddit name
  subredditId: string;           // Subreddit ID
  score: number;                 // Current score (upvotes - downvotes)
  numberOfComments: number;      // Comment count
  createdAt: Date;               // When posted
  permalink: string;             // Reddit URL path
  thumbnail?: {                  // Thumbnail (if available)
    url: string;
    width: number;
    height: number;
  };
  locked: boolean;               // Is post locked?
  removed: boolean;              // Was post removed?
  spam: boolean;                 // Marked as spam?
  nsfw: boolean;                 // NSFW flag
  spoiler: boolean;              // Spoiler flag
  distinguished?: string;        // 'moderator' | 'admin' if distinguished
  stickied: boolean;             // Is stickied/pinned?
  flair?: {                      // Post flair
    text: string;
    backgroundColor?: string;
    textColor?: string;
    templateId?: string;
  };
}
```

### Creating Posts

**Submit a text post:**
```typescript
const subreddit = await context.reddit.getCurrentSubreddit();

const post = await context.reddit.submitPost({
  title: 'Hello from Devvit!',
  subredditName: subreddit.name,
  text: 'This is a text post created by a Devvit app.',
});

console.log(`Created post: ${post.id}`);
```

**Submit a link post:**
```typescript
const post = await context.reddit.submitPost({
  title: 'Check out this link',
  subredditName: 'test',
  url: 'https://developers.reddit.com',
});
```

**Submit a custom post:**
```typescript
const post = await context.reddit.submitPost({
  title: 'Interactive Poll',
  subredditName: subreddit.name,
  preview: (
    <vstack padding="medium">
      <text>Click to view the interactive poll</text>
    </vstack>
  ),
});
```

### Modifying Posts

**Edit post (text posts only):**
```typescript
const post = await context.reddit.getPostById('t3_abc123');
await post.edit({ text: 'Updated post content' });
```

**Delete post:**
```typescript
await post.delete();
```

**Set flair:**
```typescript
await post.setFlair({
  text: 'Discussion',
  backgroundColor: '#0079D3',
  textColor: 'light',
});
```

**Lock/unlock:**
```typescript
await post.lock();
await post.unlock();
```

**Mark as NSFW/Spoiler:**
```typescript
await post.markAsNsfw();
await post.unmarkAsNsfw();
await post.markAsSpoiler();
await post.unmarkAsSpoiler();
```

**Sticky/unsticky:**
```typescript
await post.sticky();
await post.unsticky();
```

### Moderating Posts

**Approve:**
```typescript
await post.approve();
```

**Remove:**
```typescript
await post.remove();
```

**Distinguish:**
```typescript
await post.distinguish({ how: 'yes' });     // Mod distinguish
await post.distinguish({ how: 'admin' });   // Admin distinguish
await post.distinguish({ how: 'no' });      // Un-distinguish
```

**Ignore reports:**
```typescript
await post.ignoreReports();
```

## Working with Comments

### Getting Comments

**Get comment by ID:**
```typescript
const comment = await context.reddit.getCommentById('t1_xyz789');

console.log(comment.body);
console.log(comment.score);
console.log(comment.authorName);
```

**Get all comments from a post:**
```typescript
const post = await context.reddit.getPostById('t3_abc123');
const comments = await post.comments.all();

console.log(`Found ${comments.length} comments`);

comments.forEach(comment => {
  console.log(`${comment.authorName}: ${comment.body}`);
});
```

**Get top-level comments only:**
```typescript
const topLevelComments = comments.filter(c => c.parentId === post.id);
```

### Comment Properties

```typescript
interface Comment {
  id: string;                    // Fullname (e.g., 't1_xyz789')
  body: string;                  // Comment text
  authorName: string;            // Username
  authorId: string;              // User ID
  postId: string;                // Parent post ID
  parentId: string;              // Parent comment/post ID
  subredditName: string;         // Subreddit
  score: number;                 // Score
  createdAt: Date;               // When posted
  permalink: string;             // Reddit URL path
  depth: number;                 // Nesting level
  removed: boolean;              // Was removed?
  spam: boolean;                 // Marked as spam?
  distinguished?: string;        // 'moderator' | 'admin'
  stickied: boolean;             // Is stickied?
}
```

### Creating Comments

**Reply to a post:**
```typescript
const post = await context.reddit.getPostById('t3_abc123');

const comment = await context.reddit.submitComment({
  id: post.id,
  text: 'Great post! Thanks for sharing.',
});
```

**Reply to a comment:**
```typescript
const parentComment = await context.reddit.getCommentById('t1_xyz789');

const reply = await context.reddit.submitComment({
  id: parentComment.id,
  text: 'I agree with your point!',
});
```

### Modifying Comments

**Edit comment:**
```typescript
await comment.edit({ text: 'Updated comment text' });
```

**Delete comment:**
```typescript
await comment.delete();
```

### Moderating Comments

**Approve/Remove:**
```typescript
await comment.approve();
await comment.remove();
```

**Distinguish:**
```typescript
await comment.distinguish({ how: 'yes' });
```

## Working with Users

### Getting Users

**Get user by ID:**
```typescript
const user = await context.reddit.getUserById('t2_user123');

console.log(user.username);
console.log(user.karma);
console.log(user.createdAt);
```

**Get user by username:**
```typescript
const user = await context.reddit.getUserByUsername('spez');
```

**Get current user:**
```typescript
const currentUser = await context.reddit.getCurrentUser();
console.log(`Current user: ${currentUser.username}`);
```

### User Properties

```typescript
interface User {
  id: string;                    // User ID (t2_...)
  username: string;              // Username
  karma: {                       // Karma scores
    total: number;
    post: number;
    comment: number;
  };
  createdAt: Date;               // Account creation date
  nsfw: boolean;                 // NSFW profile?
  snoovatarUrl?: string;         // Avatar URL
}
```

### User Methods

**Check if user is moderator:**
```typescript
const isMod = await user.isModerator(subredditName);

if (isMod) {
  console.log(`${user.username} is a moderator`);
}
```

**Get user's posts:**
```typescript
// Not directly available through Devvit API
// You'd need to fetch subreddit posts and filter by author
```

### Moderation Actions on Users

**Ban user:**
```typescript
await context.reddit.ban({
  subredditName: 'mysubreddit',
  username: 'spammer123',
  duration: 7,                   // Days (omit for permanent)
  reason: 'Spam',
  note: 'Repeatedly posted spam links',
  message: 'You have been banned for spam',
});
```

**Unban user:**
```typescript
await context.reddit.unban({
  subredditName: 'mysubreddit',
  username: 'spammer123',
});
```

**Mute user:**
```typescript
await context.reddit.mute({
  subredditName: 'mysubreddit',
  username: 'troll456',
  note: 'Constant harassment',
});
```

**Unmute user:**
```typescript
await context.reddit.unmute({
  subredditName: 'mysubreddit',
  username: 'troll456',
});
```

**Invite as moderator:**
```typescript
await context.reddit.inviteModerator({
  subredditName: 'mysubreddit',
  username: 'trusted_user',
  permissions: ['posts', 'mail', 'wiki'],
});
```

**Remove as moderator:**
```typescript
await context.reddit.removeModerator({
  subredditName: 'mysubreddit',
  username: 'former_mod',
});
```

**Add as contributor:**
```typescript
await context.reddit.addContributor({
  subredditName: 'mysubreddit',
  username: 'contributor123',
});
```

## Working with Subreddits

### Getting Subreddits

**Get subreddit by ID:**
```typescript
const subreddit = await context.reddit.getSubredditById('t5_2qh33');
```

**Get subreddit by name:**
```typescript
const subreddit = await context.reddit.getSubredditByName('devvit');
```

**Get current subreddit:**
```typescript
const currentSub = await context.reddit.getCurrentSubreddit();
console.log(`Current subreddit: r/${currentSub.name}`);
```

### Subreddit Properties

```typescript
interface Subreddit {
  id: string;                    // Subreddit ID (t5_...)
  name: string;                  // Subreddit name (without r/)
  title: string;                 // Display title
  description: string;           // Sidebar description
  numberOfSubscribers: number;   // Subscriber count
  nsfw: boolean;                 // NSFW subreddit?
  createdAt: Date;               // When created
  type: string;                  // 'public', 'restricted', 'private'
}
```

### Subreddit Actions

**Get settings:**
```typescript
const settings = await subreddit.getSettings();

console.log(settings.title);
console.log(settings.publicDescription);
console.log(settings.allowImages);
```

**Update settings (mod only):**
```typescript
await subreddit.updateSettings({
  title: 'New Title',
  publicDescription: 'New description',
  allowImages: true,
});
```

**Get wiki page:**
```typescript
const wikiPage = await subreddit.getWikiPage('index');
console.log(wikiPage.content);
```

**Update wiki page:**
```typescript
await subreddit.updateWikiPage({
  page: 'rules',
  content: '# Subreddit Rules\n\n1. Be nice\n2. No spam',
  reason: 'Updated rules',
});
```

**Create wiki page:**
```typescript
await subreddit.createWikiPage({
  page: 'newpage',
  content: '# New Page\n\nContent here',
});
```

## Flair Management

### User Flair

**Get user flair:**
```typescript
const flair = await context.reddit.getUserFlairBySubreddit({
  subredditName: 'devvit',
  username: 'someuser',
});

console.log(flair?.text);
```

**Set user flair:**
```typescript
await context.reddit.setUserFlair({
  subredditName: 'devvit',
  username: 'someuser',
  text: 'Verified Developer',
  backgroundColor: '#0079D3',
  textColor: 'light',
});
```

**Remove user flair:**
```typescript
await context.reddit.setUserFlair({
  subredditName: 'devvit',
  username: 'someuser',
  text: '',
});
```

### Post Flair

**Get available post flairs:**
```typescript
const flairs = await subreddit.getPostFlairTemplates();

flairs.forEach(flair => {
  console.log(`${flair.text} (${flair.id})`);
});
```

**Set post flair:**
```typescript
await post.setFlair({
  text: 'Discussion',
  backgroundColor: '#0079D3',
  textColor: 'light',
});
```

**Set flair using template:**
```typescript
await post.setFlair({
  templateId: 'template-id-here',
  text: 'Custom text',
});
```

## Moderation Tools

### Modqueue

**Get modqueue items:**
```typescript
const subreddit = await context.reddit.getCurrentSubreddit();
const modQueue = await subreddit.getModQueue({
  limit: 50,
}).all();

console.log(`${modQueue.length} items in queue`);

modQueue.forEach(item => {
  if (item.type === 'post') {
    console.log(`Post: ${item.title}`);
  } else {
    console.log(`Comment: ${item.body}`);
  }
});
```

### Reports

**Get reported items:**
```typescript
const reports = await subreddit.getReports({
  limit: 25,
}).all();

reports.forEach(item => {
  console.log(`${item.type}: ${item.numReports} reports`);
});
```

### Spam

**Get spam:**
```typescript
const spam = await subreddit.getSpam({
  limit: 25,
}).all();
```

### Edited

**Get edited content:**
```typescript
const edited = await subreddit.getEdited({
  limit: 25,
}).all();
```

### Unmoderated

**Get unmoderated posts:**
```typescript
const unmoderated = await subreddit.getUnmoderated({
  limit: 25,
}).all();
```

## Search

### Search Posts

**Search in subreddit:**
```typescript
const results = await subreddit.search({
  query: 'tutorial',
  sort: 'relevance',  // 'relevance', 'hot', 'top', 'new', 'comments'
  timeframe: 'all',   // 'hour', 'day', 'week', 'month', 'year', 'all'
  limit: 25,
}).all();

results.forEach(post => {
  console.log(post.title);
});
```

## Practical Examples

### Example 1: Auto-Flair Based on Keywords

```typescript
Devvit.configure({
  redditAPI: true,
});

Devvit.addTrigger({
  event: 'PostSubmit',
  onEvent: async (event, context) => {
    const post = event.post;
    const title = post.title.toLowerCase();

    let flairText = '';
    let flairColor = '';

    if (title.includes('question') || title.includes('help')) {
      flairText = 'Question';
      flairColor = '#0079D3';
    } else if (title.includes('discussion')) {
      flairText = 'Discussion';
      flairColor = '#46D160';
    } else if (title.includes('news')) {
      flairText = 'News';
      flairColor = '#FF4500';
    }

    if (flairText) {
      await post.setFlair({
        text: flairText,
        backgroundColor: flairColor,
        textColor: 'light',
      });

      console.log(`Auto-flaired post: ${flairText}`);
    }
  },
});

export default Devvit;
```

### Example 2: Comment Statistics

```typescript
Devvit.addMenuItem({
  label: 'Analyze Comments',
  location: 'post',
  onPress: async (event, context) => {
    const post = await context.reddit.getPostById(event.targetId);
    const comments = await post.comments.all();

    if (comments.length === 0) {
      context.ui.showToast('No comments yet');
      return;
    }

    // Calculate statistics
    const totalComments = comments.length;
    const averageScore = comments.reduce((sum, c) => sum + c.score, 0) / totalComments;
    const topComment = comments.reduce((max, c) => c.score > max.score ? c : max);

    const authorCounts: { [key: string]: number } = {};
    comments.forEach(c => {
      authorCounts[c.authorName] = (authorCounts[c.authorName] || 0) + 1;
    });

    const topCommenter = Object.entries(authorCounts)
      .sort(([, a], [, b]) => b - a)[0];

    const stats = [
      `**Comment Statistics**`,
      ``,
      `📊 Total: ${totalComments}`,
      `⭐ Avg Score: ${averageScore.toFixed(1)}`,
      `🏆 Top: ${topComment.score} pts`,
      `👤 Most Active: u/${topCommenter[0]} (${topCommenter[1]} comments)`,
    ].join('\n');

    context.ui.showToast(stats);
  },
});
```

### Example 3: Bulk Approve Posts

```typescript
Devvit.addMenuItem({
  label: 'Approve Old Posts',
  location: 'subreddit',
  forUserType: 'moderator',
  onPress: async (event, context) => {
    const subreddit = await context.reddit.getCurrentSubreddit();
    const modQueue = await subreddit.getModQueue({ limit: 100 }).all();

    const oldPosts = modQueue.filter(item => {
      if (item.type !== 'post') return false;
      const ageHours = (Date.now() - item.createdAt.getTime()) / (1000 * 60 * 60);
      return ageHours > 24;
    });

    let approved = 0;
    for (const item of oldPosts) {
      try {
        await item.approve();
        approved++;
      } catch (error) {
        console.error(`Failed to approve ${item.id}:`, error);
      }
    }

    context.ui.showToast(`Approved ${approved} old posts`);
  },
});
```

### Example 4: Welcome Bot

```typescript
Devvit.configure({
  redditAPI: true,
  redis: true,
});

Devvit.addTrigger({
  event: 'PostSubmit',
  onEvent: async (event, context) => {
    const post = event.post;
    const author = event.author;

    // Check if author has posted before
    const hasPostedKey = `hasPosted_${author.id}_${post.subredditName}`;
    const hasPosted = await context.redis.get(hasPostedKey);

    if (!hasPosted) {
      // First post in this subreddit
      await context.reddit.submitComment({
        id: post.id,
        text: `Welcome to r/${post.subredditName}, u/${author.username}! 👋\n\nThank you for your first post. Please read our rules in the sidebar!`,
      });

      // Mark as posted
      await context.redis.set(hasPostedKey, 'true');
    }
  },
});

export default Devvit;
```

### Example 5: Report Analysis

```typescript
Devvit.addMenuItem({
  label: 'Report Summary',
  location: 'subreddit',
  forUserType: 'moderator',
  onPress: async (event, context) => {
    const subreddit = await context.reddit.getCurrentSubreddit();
    const reports = await subreddit.getReports({ limit: 100 }).all();

    if (reports.length === 0) {
      context.ui.showToast('No reports');
      return;
    }

    const postReports = reports.filter(r => r.type === 'post').length;
    const commentReports = reports.filter(r => r.type === 'comment').length;
    const totalReportCount = reports.reduce((sum, r) => sum + r.numReports, 0);

    const summary = [
      `**Report Summary**`,
      ``,
      `📝 Total Items: ${reports.length}`,
      `📄 Posts: ${postReports}`,
      `💬 Comments: ${commentReports}`,
      `🚩 Total Reports: ${totalReportCount}`,
    ].join('\n');

    context.ui.showToast(summary);
  },
});
```

## Best Practices

### 1. Handle Rate Limits

Reddit API has rate limits. Batch operations carefully:

```typescript
// Bad: Sequential, slow, hits rate limits
for (const post of posts) {
  await post.approve();
}

// Good: Parallel with limit
const chunks = [];
for (let i = 0; i < posts.length; i += 10) {
  chunks.push(posts.slice(i, i + 10));
}

for (const chunk of chunks) {
  await Promise.all(chunk.map(p => p.approve()));
  // Optional: add delay between chunks
  await new Promise(resolve => setTimeout(resolve, 1000));
}
```

### 2. Check Permissions

Always verify permissions before moderation actions:

```typescript
const user = await context.reddit.getCurrentUser();
const subreddit = await context.reddit.getCurrentSubreddit();
const isMod = await user.isModerator(subreddit.name);

if (!isMod) {
  context.ui.showToast('You must be a moderator');
  return;
}

// Proceed with mod action
```

### 3. Handle Deleted Content

```typescript
try {
  const post = await context.reddit.getPostById(postId);

  if (post.removed) {
    context.ui.showToast('This post was removed');
    return;
  }

  // Process post
} catch (error) {
  context.ui.showToast('Post not found or deleted');
}
```

### 4. Validate Data

```typescript
const post = await context.reddit.getPostById(postId);

// Check if it's a text post before editing
if (!post.body) {
  context.ui.showToast('Can only edit text posts');
  return;
}

await post.edit({ text: 'New content' });
```

## Key Takeaways

1. **context.reddit** - Gateway to all Reddit operations
2. **Posts, comments, users, subreddits** - Four main object types
3. **Moderation actions** - Approve, remove, distinguish, ban, etc.
4. **Async operations** - All API calls are async
5. **Error handling** - Always wrap in try-catch
6. **Permissions** - Check before moderation actions
7. **Rate limits** - Batch operations, use parallel requests wisely

## Checkpoint Questions

1. How do you get the current subreddit?
2. What's the difference between `post.remove()` and `post.delete()`?
3. How do you get all comments from a post?
4. What method checks if a user is a moderator?
5. How do you ban a user for 7 days?

<details>
<summary>Click to see answers</summary>

1. `await context.reddit.getCurrentSubreddit()`
2. `remove()` is a mod action (can be undone), `delete()` permanently deletes (only by author)
3. `const comments = await post.comments.all()`
4. `await user.isModerator(subredditName)`
5. `await context.reddit.ban({ subredditName: 'sub', username: 'user', duration: 7, reason: 'reason' })`

</details>

## Additional Resources

- [Reddit API Documentation](https://developers.reddit.com/docs/api)
- [Reddit Data Models](https://developers.reddit.com/docs/api/models)
- [Moderation API](https://developers.reddit.com/docs/api/moderation)

---

➡️ **Continue to [Lesson 7: State Management and Storage](./07-storage.md)**

**Estimated time to complete**: 2 hours
**Practice exercises**: Multiple examples included
**Prerequisites**: Lessons 1-5 completed
