# Lesson 8: Scheduler and Background Jobs

**Duration**: 1.5 hours
**Level**: Intermediate

## Overview

Learn how to schedule recurring tasks and run background jobs in Devvit. Perfect for automated posts, cleanup tasks, reminders, and periodic data processing.

## The Scheduler

Devvit's scheduler allows you to:
- Run tasks at specific times
- Schedule recurring tasks (cron-style)
- Defer heavy work to background jobs
- Create automated posting schedules

## Enabling the Scheduler

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
  redis: true,  // Often needed with scheduler
});

export default Devvit;
```

Note: The scheduler is always available; no special configuration needed!

## Scheduled Jobs (Recurring)

Create jobs that run on a schedule using cron syntax.

### Basic Scheduled Job

```typescript
Devvit.addSchedulerJob({
  name: 'dailyTask',
  cron: '0 9 * * *',  // Every day at 9:00 AM UTC
  onRun: async (event, context) => {
    console.log('Daily task running!');

    const subreddit = await context.reddit.getCurrentSubreddit();
    await context.reddit.submitPost({
      title: 'Daily Discussion Thread',
      subredditName: subreddit.name,
      text: `Welcome to today's daily discussion!`,
    });
  },
});
```

### Cron Syntax

Cron expressions have 5 fields:

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6) (Sunday to Saturday)
│ │ │ │ │
* * * * *
```

### Common Cron Examples

```typescript
// Every minute
'* * * * *'

// Every hour at minute 0
'0 * * * *'

// Every day at midnight UTC
'0 0 * * *'

// Every day at 9 AM UTC
'0 9 * * *'

// Every Monday at 9 AM
'0 9 * * 1'

// Every 15 minutes
'*/15 * * * *'

// Every 6 hours
'0 */6 * * *'

// First day of every month at midnight
'0 0 1 * *'

// Weekdays at 9 AM
'0 9 * * 1-5'

// Twice a day (9 AM and 5 PM)
'0 9,17 * * *'
```

### Multiple Scheduled Jobs

```typescript
// Daily stats at midnight
Devvit.addSchedulerJob({
  name: 'dailyStats',
  cron: '0 0 * * *',
  onRun: async (event, context) => {
    console.log('Generating daily stats...');
    // Generate and post statistics
  },
});

// Weekly reminder every Monday
Devvit.addSchedulerJob({
  name: 'weeklyReminder',
  cron: '0 9 * * 1',
  onRun: async (event, context) => {
    console.log('Posting weekly reminder...');
    // Post weekly thread
  },
});

// Cleanup every hour
Devvit.addSchedulerJob({
  name: 'hourlyCleanup',
  cron: '0 * * * *',
  onRun: async (event, context) => {
    console.log('Running cleanup...');
    // Clean up old data
  },
});
```

## One-Time Jobs

Schedule a job to run once at a specific time.

### Schedule a Future Job

```typescript
Devvit.addMenuItem({
  label: 'Remind Me in 1 Hour',
  location: 'post',
  onPress: async (event, context) => {
    const post = await context.reddit.getPostById(event.targetId);
    const user = await context.reddit.getCurrentUser();

    // Schedule job for 1 hour from now
    await context.scheduler.runJob({
      name: 'reminderJob',
      data: {
        postId: post.id,
        userId: user.id,
        username: user.username,
      },
      runAt: new Date(Date.now() + 3600000), // 1 hour
    });

    context.ui.showToast('Reminder set for 1 hour!');
  },
});

// Define the job handler
Devvit.addSchedulerJob({
  name: 'reminderJob',
  onRun: async (event, context) => {
    const { postId, username } = event.data as {
      postId: string;
      username: string;
    };

    const post = await context.reddit.getPostById(postId);

    await context.reddit.submitComment({
      id: postId,
      text: `u/${username}, this is your reminder about: "${post.title}"`,
    });
  },
});
```

### Passing Data to Jobs

```typescript
await context.scheduler.runJob({
  name: 'processData',
  data: {
    userId: 'abc123',
    action: 'cleanup',
    threshold: 100,
    items: ['item1', 'item2'],
  },
  runAt: new Date(Date.now() + 60000), // 1 minute
});

Devvit.addSchedulerJob({
  name: 'processData',
  onRun: async (event, context) => {
    const { userId, action, threshold, items } = event.data as {
      userId: string;
      action: string;
      threshold: number;
      items: string[];
    };

    console.log(`Processing ${action} for user ${userId}`);
    // Use the data
  },
});
```

## Complete Examples

### Example 1: Daily Discussion Thread

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
  redis: true,
});

Devvit.addSchedulerJob({
  name: 'dailyThread',
  cron: '0 9 * * *', // 9 AM UTC daily
  onRun: async (event, context) => {
    const subreddit = await context.reddit.getCurrentSubreddit();

    // Get current date
    const now = new Date();
    const dateStr = now.toLocaleDateString('en-US', {
      weekday: 'long',
      year: 'numeric',
      month: 'long',
      day: 'numeric',
    });

    // Create the post
    const post = await context.reddit.submitPost({
      title: `Daily Discussion Thread - ${dateStr}`,
      subredditName: subreddit.name,
      text: [
        `Welcome to today's daily discussion thread!`,
        ``,
        `Feel free to discuss anything here.`,
        ``,
        `**Rules:**`,
        `- Be respectful`,
        `- Stay on topic`,
        `- Have fun!`,
      ].join('\n'),
    });

    // Sticky the post
    await post.sticky();

    // Track in Redis
    await context.redis.set('last-daily-thread', post.id);
    await context.redis.incrBy('total-daily-threads', 1);

    console.log(`Created daily thread: ${post.id}`);
  },
});

export default Devvit;
```

### Example 2: Reminder System

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
});

// Form to create reminder
const reminderForm = Devvit.createForm(
  {
    title: 'Set Reminder',
    fields: [
      {
        name: 'duration',
        label: 'Remind me in...',
        type: 'select',
        options: [
          { label: '1 hour', value: '3600000' },
          { label: '6 hours', value: '21600000' },
          { label: '24 hours', value: '86400000' },
          { label: '1 week', value: '604800000' },
        ],
        required: true,
      },
      {
        name: 'message',
        label: 'Reminder message',
        type: 'paragraph',
        required: false,
        helpText: 'Optional custom message',
      },
    ],
  },
  async (event, context) => {
    const duration = Number(event.values.duration);
    const message = event.values.message as string | undefined;
    const user = await context.reddit.getCurrentUser();

    await context.scheduler.runJob({
      name: 'sendReminder',
      data: {
        username: user.username,
        message: message || 'This is your reminder!',
      },
      runAt: new Date(Date.now() + duration),
    });

    const hours = duration / 3600000;
    context.ui.showToast(`Reminder set for ${hours} hour(s) from now`);
  }
);

Devvit.addMenuItem({
  label: 'Set Reminder',
  location: 'subreddit',
  onPress: async (event, context) => {
    context.ui.showForm(reminderForm);
  },
});

Devvit.addSchedulerJob({
  name: 'sendReminder',
  onRun: async (event, context) => {
    const { username, message } = event.data as {
      username: string;
      message: string;
    };

    const subreddit = await context.reddit.getCurrentSubreddit();

    // Send message by creating a post mentioning the user
    await context.reddit.submitPost({
      title: `Reminder for u/${username}`,
      subredditName: subreddit.name,
      text: `u/${username}, ${message}`,
    });

    console.log(`Sent reminder to ${username}`);
  },
});

export default Devvit;
```

### Example 3: Automated Content Cleanup

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
  redis: true,
});

// Run cleanup every 6 hours
Devvit.addSchedulerJob({
  name: 'cleanupOldPosts',
  cron: '0 */6 * * *',
  onRun: async (event, context) => {
    const subreddit = await context.reddit.getCurrentSubreddit();

    // Get new posts
    const posts = await subreddit.getNewPosts({ limit: 100 }).all();

    let removedCount = 0;
    const maxAgeMs = 7 * 24 * 60 * 60 * 1000; // 7 days

    for (const post of posts) {
      const age = Date.now() - post.createdAt.getTime();

      // Remove posts older than 7 days with score < 5
      if (age > maxAgeMs && post.score < 5) {
        try {
          await post.remove();
          removedCount++;
          console.log(`Removed old post: ${post.id}`);
        } catch (error) {
          console.error(`Failed to remove ${post.id}:`, error);
        }
      }
    }

    // Track statistics
    await context.redis.set('last-cleanup', new Date().toISOString());
    await context.redis.incrBy('total-removed', removedCount);

    console.log(`Cleanup complete: removed ${removedCount} posts`);
  },
});

// Menu to view cleanup stats
Devvit.addMenuItem({
  label: 'Cleanup Stats',
  location: 'subreddit',
  forUserType: 'moderator',
  onPress: async (event, context) => {
    const lastCleanup = await context.redis.get('last-cleanup');
    const totalRemoved = await context.redis.get('total-removed') || '0';

    const stats = [
      `**Cleanup Statistics**`,
      ``,
      `Last run: ${lastCleanup || 'Never'}`,
      `Total removed: ${totalRemoved}`,
    ].join('\n');

    context.ui.showToast(stats);
  },
});

export default Devvit;
```

### Example 4: Weekly Leaderboard

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
  redis: true,
});

interface UserScore {
  username: string;
  score: number;
}

// Track post submissions
Devvit.addTrigger({
  event: 'PostSubmit',
  onEvent: async (event, context) => {
    const author = event.author;

    // Increment weekly score
    const key = `weekly:${author.username}`;
    await context.redis.incrBy(key, 10); // 10 points per post

    // Set expiration for next week
    await context.redis.expire(key, new Date(Date.now() + 7 * 24 * 60 * 60 * 1000));
  },
});

// Track comment submissions
Devvit.addTrigger({
  event: 'CommentSubmit',
  onEvent: async (event, context) => {
    const author = event.author;

    const key = `weekly:${author.username}`;
    await context.redis.incrBy(key, 1); // 1 point per comment
  },
});

// Post weekly leaderboard every Monday at 9 AM
Devvit.addSchedulerJob({
  name: 'weeklyLeaderboard',
  cron: '0 9 * * 1',
  onRun: async (event, context) => {
    const subreddit = await context.reddit.getCurrentSubreddit();

    // Collect scores (you'd need to track usernames separately)
    // For simplicity, assume we have a list
    const leaderboardData: UserScore[] = []; // Populate from Redis

    // Sort by score
    leaderboardData.sort((a, b) => b.score - a.score);

    // Build leaderboard post
    const lines = [
      `# Weekly Leaderboard`,
      ``,
      `Congratulations to our top contributors this week!`,
      ``,
      `## Top 10`,
    ];

    leaderboardData.slice(0, 10).forEach((user, index) => {
      const medal = index === 0 ? '🥇' : index === 1 ? '🥈' : index === 2 ? '🥉' : '';
      lines.push(`${index + 1}. ${medal} u/${user.username} - ${user.score} points`);
    });

    const post = await context.reddit.submitPost({
      title: 'Weekly Leaderboard',
      subredditName: subreddit.name,
      text: lines.join('\n'),
    });

    await post.sticky();

    // Clear weekly scores
    for (const user of leaderboardData) {
      await context.redis.del(`weekly:${user.username}`);
    }

    console.log('Posted weekly leaderboard');
  },
});

export default Devvit;
```

### Example 5: Backup System

```typescript
Devvit.configure({
  redditAPI: true,
  redis: true,
});

// Backup data every day at midnight
Devvit.addSchedulerJob({
  name: 'dailyBackup',
  cron: '0 0 * * *',
  onRun: async (event, context) => {
    // Get all data to backup
    const config = await context.redis.get('app:config');
    const stats = await context.redis.get('app:stats');
    const leaderboard = await context.redis.get('leaderboard');

    // Create backup object
    const backup = {
      timestamp: Date.now(),
      data: {
        config,
        stats,
        leaderboard,
      },
    };

    // Store backup with date
    const dateStr = new Date().toISOString().split('T')[0];
    await context.redis.set(`backup:${dateStr}`, JSON.stringify(backup));

    // Keep only last 7 backups
    // (You'd implement cleanup logic here)

    console.log(`Backup created: ${dateStr}`);
  },
});

// Menu to restore from backup
Devvit.addMenuItem({
  label: 'Restore Backup',
  location: 'subreddit',
  forUserType: 'moderator',
  onPress: async (event, context) => {
    const today = new Date().toISOString().split('T')[0];
    const backupStr = await context.redis.get(`backup:${today}`);

    if (!backupStr) {
      context.ui.showToast('No backup found for today');
      return;
    }

    const backup = JSON.parse(backupStr);

    // Restore data
    if (backup.data.config) {
      await context.redis.set('app:config', backup.data.config);
    }
    if (backup.data.stats) {
      await context.redis.set('app:stats', backup.data.stats);
    }
    if (backup.data.leaderboard) {
      await context.redis.set('leaderboard', backup.data.leaderboard);
    }

    context.ui.showToast('Backup restored!');
  },
});
```

## Best Practices

### 1. Keep Jobs Short

Jobs have time limits, so keep them focused:

```typescript
// Bad: Too much work
Devvit.addSchedulerJob({
  name: 'heavyJob',
  cron: '0 * * * *',
  onRun: async (event, context) => {
    const posts = await subreddit.getNewPosts({ limit: 1000 }).all();

    for (const post of posts) {
      // This could timeout
      await processPost(post);
    }
  },
});

// Good: Process in chunks
Devvit.addSchedulerJob({
  name: 'efficientJob',
  cron: '0 * * * *',
  onRun: async (event, context) => {
    const posts = await subreddit.getNewPosts({ limit: 50 }).all();

    // Process limited batch
    await Promise.all(posts.map(p => processPost(p)));
  },
});
```

### 2. Use Redis for Job State

Track job progress:

```typescript
Devvit.addSchedulerJob({
  name: 'statefulJob',
  cron: '0 0 * * *',
  onRun: async (event, context) => {
    // Record job start
    await context.redis.set('job:last-run', new Date().toISOString());

    try {
      // Do work
      await doWork(context);

      // Record success
      await context.redis.set('job:last-success', new Date().toISOString());
      await context.redis.incrBy('job:success-count', 1);
    } catch (error) {
      // Record failure
      await context.redis.set('job:last-error', String(error));
      await context.redis.incrBy('job:error-count', 1);
    }
  },
});
```

### 3. Handle Errors Gracefully

```typescript
Devvit.addSchedulerJob({
  name: 'robustJob',
  cron: '0 * * * *',
  onRun: async (event, context) => {
    try {
      await performTask(context);
    } catch (error) {
      console.error('Job failed:', error);

      // Don't crash, log error
      await context.redis.set('last-job-error', JSON.stringify({
        timestamp: Date.now(),
        error: String(error),
      }));
    }
  },
});
```

### 4. Test Cron Expressions

Use online tools to verify cron syntax:
- [Crontab Guru](https://crontab.guru/)
- Test in development first

### 5. Use UTC Time

All cron jobs run in UTC. Account for timezones:

```typescript
// If you want 9 AM Eastern Time:
// EST is UTC-5, so 9 AM EST = 2 PM UTC
'0 14 * * *'  // 2 PM UTC = 9 AM EST

// EDT is UTC-4, so 9 AM EDT = 1 PM UTC
'0 13 * * *'  // 1 PM UTC = 9 AM EDT
```

### 6. Idempotent Jobs

Design jobs to be safely re-run:

```typescript
Devvit.addSchedulerJob({
  name: 'idempotentJob',
  cron: '0 0 * * *',
  onRun: async (event, context) => {
    // Check if already ran today
    const today = new Date().toISOString().split('T')[0];
    const lastRun = await context.redis.get('daily-job-date');

    if (lastRun === today) {
      console.log('Job already ran today');
      return;
    }

    // Do work
    await doWork(context);

    // Mark as complete
    await context.redis.set('daily-job-date', today);
  },
});
```

## Debugging Scheduled Jobs

### View job logs:

```bash
devvit logs <app-name>
```

### Manual job trigger:

Create a menu item to manually run a job:

```typescript
Devvit.addMenuItem({
  label: 'Run Daily Job Now',
  location: 'subreddit',
  forUserType: 'moderator',
  onPress: async (event, context) => {
    await context.scheduler.runJob({
      name: 'dailyTask',
      data: {},
      runAt: new Date(), // Run immediately
    });

    context.ui.showToast('Job triggered!');
  },
});
```

### Job status dashboard:

```typescript
Devvit.addMenuItem({
  label: 'Job Status',
  location: 'subreddit',
  forUserType: 'moderator',
  onPress: async (event, context) => {
    const lastRun = await context.redis.get('job:last-run');
    const lastSuccess = await context.redis.get('job:last-success');
    const successCount = await context.redis.get('job:success-count') || '0';
    const errorCount = await context.redis.get('job:error-count') || '0';

    const status = [
      `**Job Status**`,
      ``,
      `Last run: ${lastRun || 'Never'}`,
      `Last success: ${lastSuccess || 'Never'}`,
      `Success count: ${successCount}`,
      `Error count: ${errorCount}`,
    ].join('\n');

    context.ui.showToast(status);
  },
});
```

## Key Takeaways

1. **Two job types** - Scheduled (recurring) and one-time
2. **Cron syntax** - Five fields for scheduling
3. **UTC timezone** - All times are UTC
4. **Pass data** - Jobs can receive custom data
5. **Keep short** - Jobs have time limits
6. **Error handling** - Always wrap in try-catch
7. **Track state** - Use Redis to monitor jobs
8. **Idempotent** - Design jobs to be safely re-run

## Checkpoint Questions

1. What cron expression runs every day at midnight?
2. How do you schedule a one-time job?
3. What timezone do scheduled jobs use?
4. How do you pass data to a job?
5. How can you manually trigger a job for testing?

<details>
<summary>Click to see answers</summary>

1. `'0 0 * * *'`
2. `await context.scheduler.runJob({ name: 'jobName', data: {}, runAt: new Date(timestamp) })`
3. UTC (Coordinated Universal Time)
4. Use the `data` parameter: `runJob({ name: 'job', data: { key: 'value' }, runAt: date })`
5. Create a menu item that calls `context.scheduler.runJob()` with `runAt: new Date()`

</details>

## Practice Exercises

### Exercise 1: Weekly Recap

Create a job that posts a weekly recap every Sunday with stats from the past week.

### Exercise 2: Timed Reminder

Build a system where users can set reminders for specific future times with custom messages.

### Exercise 3: Auto-Archive

Create a job that automatically archives (locks) posts older than 30 days.

## Additional Resources

- [Crontab Guru](https://crontab.guru/) - Cron expression editor
- [Devvit Scheduler Documentation](https://developers.reddit.com/docs/scheduler)
- [UTC Time Converter](https://www.worldtimebuddy.com/)

---

➡️ **Continue to [Lesson 9: Advanced Features and Best Practices](./09-advanced.md)**

**Estimated time to complete**: 1.5 hours
**Practice exercises**: 3 suggested projects
**Prerequisites**: Lessons 1-7 completed
