# Lesson 10: Testing, Debugging, and Deployment

**Duration**: 1.5 hours
**Level**: Advanced

## Overview

Learn how to test your Devvit apps, debug issues, and deploy to production. By the end, you'll be able to publish fully-tested, production-ready apps.

## Local Development and Testing

### Development Server

Start your local dev environment:

```bash
devvit dev
```

This:
- Compiles your TypeScript code
- Starts a local server at `http://localhost:3000`
- Opens the Devvit Playground in your browser
- Watches for file changes and hot-reloads

### The Playground

The playground provides:
- Simulated Reddit environment
- Test different user types (regular user, moderator)
- Trigger menu actions
- View custom posts
- Test forms

**Limitations:**
- Not a real subreddit
- Limited Reddit API functionality
- Some triggers may not work
- No real scheduling

### Manual Testing Checklist

Test each feature:

- [ ] Menu actions appear and function
- [ ] Forms display correctly and validate input
- [ ] Error messages are user-friendly
- [ ] Toast notifications work
- [ ] Custom posts render properly
- [ ] Data persists in Redis
- [ ] Triggers fire correctly
- [ ] Scheduled jobs are configured

## Debugging Techniques

### 1. Console Logging

The most basic debugging tool:

```typescript
console.log('Debug:', variable);
console.log('Post data:', JSON.stringify(post, null, 2));
console.error('Error occurred:', error);
console.warn('Warning:', message);
```

**View logs:**

**Local development:**
```bash
# Logs appear in terminal running devvit dev
```

**Production:**
```bash
devvit logs <app-name>

# Follow logs in real-time
devvit logs <app-name> --follow

# Filter by time
devvit logs <app-name> --since 1h
```

### 2. Debug Menu Items

Create debug menu items for testing:

```typescript
if (DEBUG_MODE) {
  Devvit.addMenuItem({
    label: '[DEBUG] View Storage',
    location: 'subreddit',
    forUserType: 'moderator',
    onPress: async (event, context) => {
      const config = await context.redis.get('app:config');
      const stats = await context.redis.get('app:stats');

      console.log('Config:', config);
      console.log('Stats:', stats);

      context.ui.showToast('Check console for data');
    },
  });

  Devvit.addMenuItem({
    label: '[DEBUG] Clear Data',
    location: 'subreddit',
    forUserType: 'moderator',
    onPress: async (event, context) => {
      await context.redis.del(['app:config', 'app:stats', 'counter']);
      context.ui.showToast('Data cleared');
    },
  });
}
```

### 3. Error Tracking

Track errors in Redis:

```typescript
async function logError(context: Context, error: Error, metadata?: any) {
  const errorLog = {
    timestamp: Date.now(),
    message: error.message,
    stack: error.stack,
    metadata,
  };

  // Store last 10 errors
  const errorsStr = await context.redis.get('app:errors');
  const errors = JSON.parse(errorsStr || '[]');
  errors.unshift(errorLog);
  const limited = errors.slice(0, 10);

  await context.redis.set('app:errors', JSON.stringify(limited));
}

Devvit.addMenuItem({
  label: 'Action',
  location: 'post',
  onPress: async (event, context) => {
    try {
      // Your code
    } catch (error) {
      await logError(context, error as Error, { postId: event.targetId });
      context.ui.showToast('Error occurred');
    }
  },
});
```

### 4. Debugging Custom Posts

Add debug info to custom posts:

```typescript
Devvit.addCustomPostType({
  name: 'my-post',
  height: 'regular',
  render: (context) => {
    const [data, setData] = useState({ value: 0 });

    return (
      <vstack padding="medium" gap="small">
        <text>Value: {data.value}</text>

        {/* Debug info */}
        <text size="xsmall" color="neutral-content-weak">
          Debug: postId={context.postId}
        </text>

        <button onPress={() => {
          console.log('Button clicked, current value:', data.value);
          setData({ value: data.value + 1 });
        }}>
          Increment
        </button>
      </vstack>
    );
  },
});
```

### 5. TypeScript Type Checking

Run type checker without building:

```bash
npx tsc --noEmit
```

Add to `package.json`:
```json
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "dev": "devvit dev",
    "build": "devvit build"
  }
}
```

Run: `npm run typecheck`

## Common Issues and Solutions

### Issue 1: Handler Not Responding

**Symptoms:** Menu item doesn't appear or does nothing

**Debugging:**
```typescript
Devvit.addMenuItem({
  label: 'Test Action',
  location: 'post',
  onPress: async (event, context) => {
    console.log('Handler called!', event);

    try {
      // Your code
      console.log('Handler completed successfully');
    } catch (error) {
      console.error('Handler error:', error);
    }
  },
});
```

**Solutions:**
- Check `export default Devvit;` is present
- Verify configuration options are enabled
- Check for JavaScript syntax errors
- Restart dev server

### Issue 2: Data Not Persisting

**Symptoms:** Redis data disappears

**Debugging:**
```typescript
Devvit.addMenuItem({
  label: 'Test Storage',
  location: 'subreddit',
  onPress: async (event, context) => {
    console.log('Writing to Redis...');
    await context.redis.set('test-key', 'test-value');

    console.log('Reading from Redis...');
    const value = await context.redis.get('test-key');
    console.log('Read value:', value);

    context.ui.showToast(`Value: ${value}`);
  },
});
```

**Solutions:**
- Ensure `redis: true` in configuration
- Check key names are consistent
- Verify no expiration is set unintentionally
- Check for typos in key names

### Issue 3: Triggers Not Firing

**Symptoms:** PostSubmit, CommentSubmit events not working

**Debugging:**
```typescript
Devvit.addTrigger({
  event: 'PostSubmit',
  onEvent: async (event, context) => {
    console.log('PostSubmit triggered!', event.post.id);

    try {
      // Your logic
    } catch (error) {
      console.error('Trigger error:', error);
    }
  },
});
```

**Solutions:**
- Test in a real subreddit (triggers don't work in playground)
- Check app is installed on the subreddit
- Verify Reddit API is enabled
- Check for errors in logs

### Issue 4: Timeout Errors

**Symptoms:** Function times out

**Solutions:**
- Reduce work per execution
- Use parallel operations (`Promise.all()`)
- Move heavy work to scheduled jobs
- Limit batch sizes

```typescript
// Bad: Too slow
for (const item of items) {
  await processItem(item);
}

// Good: Parallel processing
await Promise.all(items.map(item => processItem(item)));
```

### Issue 5: Custom Post Not Rendering

**Debugging:**
```typescript
Devvit.addCustomPostType({
  name: 'my-post',
  height: 'regular',
  render: (context) => {
    console.log('Render called');

    try {
      return (
        <vstack>
          <text>Hello</text>
        </vstack>
      );
    } catch (error) {
      console.error('Render error:', error);
      return (
        <vstack>
          <text color="error">Error rendering</text>
        </vstack>
      );
    }
  },
});
```

**Solutions:**
- Check JSX syntax
- Verify all components are properly closed
- Use try-catch in render
- Check for undefined values

## Pre-Deployment Checklist

Before deploying to production:

### Code Quality
- [ ] All TypeScript errors resolved
- [ ] No console.error messages in normal operation
- [ ] All features tested manually
- [ ] Error handling for all async operations
- [ ] Input validation on all forms
- [ ] Rate limiting implemented where needed

### Configuration
- [ ] `devvit.yaml` version updated
- [ ] Settings documented in YAML
- [ ] Secrets marked with `isSecret: true`
- [ ] App description is clear

### Performance
- [ ] Expensive operations cached
- [ ] Batch operations optimized
- [ ] No infinite loops
- [ ] Redis keys have reasonable TTL

### Security
- [ ] User input validated and sanitized
- [ ] Permissions checked before mod actions
- [ ] No secrets in code or logs
- [ ] Rate limiting on user actions

### Documentation
- [ ] README with setup instructions
- [ ] App description clear
- [ ] Settings explained
- [ ] Known issues documented

## Deployment Process

### 1. Build Your App

```bash
devvit build
```

This:
- Compiles TypeScript
- Bundles your code
- Validates configuration
- Creates deployable package

**Fix any errors before proceeding.**

### 2. Upload to Reddit

```bash
devvit upload
```

This:
- Uploads your app to Reddit's servers
- Makes it available for installation
- Returns app version info

**Note:** Upload creates a new version but doesn't install it anywhere.

### 3. Install on Subreddit

```bash
devvit install <subreddit-name>
```

Or install via Reddit:
1. Go to your subreddit
2. Mod Tools → Apps
3. Find your app
4. Click "Install"

### 4. Configure Settings

After installation:
1. Go to Mod Tools → Apps
2. Click on your app
3. Configure settings (API keys, thresholds, etc.)
4. Save

### 5. Test in Production

**Thorough testing:**
- Create test posts/comments
- Try all menu actions
- Test forms
- Verify triggers fire
- Check scheduled jobs
- Monitor logs: `devvit logs <app-name> --follow`

### 6. Monitor and Iterate

Watch for issues:

```bash
# View recent logs
devvit logs <app-name>

# Follow logs live
devvit logs <app-name> --follow

# Check last hour
devvit logs <app-name> --since 1h
```

## Versioning

Update version in `devvit.yaml`:

```yaml
name: my-app
version: 1.0.1  # Increment for each release
```

**Semantic Versioning:**
- `1.0.0` → `1.0.1` - Bug fixes
- `1.0.0` → `1.1.0` - New features
- `1.0.0` → `2.0.0` - Breaking changes

## Updating Your App

### Make Changes

1. Update your code
2. Test locally: `devvit dev`
3. Increment version in `devvit.yaml`
4. Build: `devvit build`

### Deploy Update

```bash
# Upload new version
devvit upload

# Update installed apps
devvit install <subreddit-name>
```

**Note:** Updates are not automatic. Each subreddit must update manually.

## Publishing to Reddit App Directory

Once your app is stable, publish it for others:

### Requirements

1. **Thorough testing** - App must be stable
2. **Good documentation** - Clear README and settings
3. **User-friendly** - Intuitive interface
4. **No known critical bugs**
5. **Follows Reddit's policies**

### Submission Process

1. Polish your app
2. Create comprehensive README
3. Add screenshots/demos
4. Submit via [Reddit Developer Portal](https://developers.reddit.com)
5. Wait for review

### App Store Guidelines

- Clear, descriptive name
- Accurate description
- Privacy policy (if collecting data)
- Support contact
- Regular updates and maintenance

## Best Practices for Production

### 1. Logging Strategy

```typescript
const LogLevel = {
  DEBUG: 0,
  INFO: 1,
  WARN: 2,
  ERROR: 3,
} as const;

const CURRENT_LOG_LEVEL = LogLevel.INFO;

function log(level: number, ...args: any[]) {
  if (level >= CURRENT_LOG_LEVEL) {
    console.log(...args);
  }
}

// Usage
log(LogLevel.DEBUG, 'Debug info');  // Only in dev
log(LogLevel.INFO, 'Operation completed');
log(LogLevel.WARN, 'Warning:', warning);
log(LogLevel.ERROR, 'Error:', error);
```

### 2. Feature Flags

```typescript
const FEATURES = {
  NEW_ALGORITHM: false,
  BETA_UI: false,
  ADVANCED_STATS: true,
};

Devvit.addMenuItem({
  label: 'New Feature',
  location: 'post',
  onPress: async (event, context) => {
    if (!FEATURES.NEW_ALGORITHM) {
      context.ui.showToast('Feature not available');
      return;
    }

    // New feature code
  },
});
```

### 3. Graceful Degradation

```typescript
async function getStats(context: Context): Promise<Stats> {
  try {
    // Try primary method
    return await fetchLiveStats(context);
  } catch (error) {
    console.warn('Live stats failed, using cache:', error);

    try {
      // Fall back to cache
      const cached = await context.redis.get('cached-stats');
      if (cached) {
        return JSON.parse(cached);
      }
    } catch {
      // Ignore cache errors
    }

    // Return defaults
    return { count: 0, updated: Date.now() };
  }
}
```

### 4. Health Checks

```typescript
Devvit.addMenuItem({
  label: '[ADMIN] Health Check',
  location: 'subreddit',
  forUserType: 'moderator',
  onPress: async (event, context) => {
    const checks = {
      redis: false,
      reddit: false,
      settings: false,
    };

    // Test Redis
    try {
      await context.redis.set('health-check', 'ok');
      const value = await context.redis.get('health-check');
      checks.redis = value === 'ok';
      await context.redis.del('health-check');
    } catch {
      checks.redis = false;
    }

    // Test Reddit API
    try {
      await context.reddit.getCurrentSubreddit();
      checks.reddit = true;
    } catch {
      checks.reddit = false;
    }

    // Test Settings
    try {
      await context.settings.get('enabled');
      checks.settings = true;
    } catch {
      checks.settings = false;
    }

    const status = [
      `**Health Check**`,
      ``,
      `Redis: ${checks.redis ? '✅' : '❌'}`,
      `Reddit API: ${checks.reddit ? '✅' : '❌'}`,
      `Settings: ${checks.settings ? '✅' : '❌'}`,
    ].join('\n');

    context.ui.showToast(status);
  },
});
```

### 5. User Feedback

Always provide clear feedback:

```typescript
Devvit.addMenuItem({
  label: 'Process',
  location: 'post',
  onPress: async (event, context) => {
    // Show processing state
    context.ui.showToast('Processing...');

    try {
      await longOperation();

      // Success feedback
      context.ui.showToast({
        text: '✅ Completed successfully!',
        appearance: 'success',
      });
    } catch (error) {
      // Error feedback
      context.ui.showToast({
        text: '❌ Operation failed. Please try again.',
        appearance: 'error',
      });
    }
  },
});
```

## Monitoring Production Apps

### Key Metrics to Track

Store in Redis:
- Total executions
- Success/error rates
- User engagement
- Performance metrics

```typescript
async function trackMetrics(context: Context, metric: string, value: number = 1) {
  const date = new Date().toISOString().split('T')[0];
  const key = `metrics:${date}:${metric}`;
  await context.redis.incrBy(key, value);

  // Keep metrics for 30 days
  await context.redis.expire(key, new Date(Date.now() + 30 * 24 * 60 * 60 * 1000));
}

Devvit.addMenuItem({
  label: 'Action',
  location: 'post',
  onPress: async (event, context) => {
    await trackMetrics(context, 'action-clicks');

    try {
      // Your code
      await trackMetrics(context, 'action-success');
    } catch (error) {
      await trackMetrics(context, 'action-errors');
    }
  },
});
```

### View Metrics

```typescript
Devvit.addMenuItem({
  label: 'View Metrics',
  location: 'subreddit',
  forUserType: 'moderator',
  onPress: async (event, context) => {
    const date = new Date().toISOString().split('T')[0];

    const clicks = await context.redis.get(`metrics:${date}:action-clicks`) || '0';
    const success = await context.redis.get(`metrics:${date}:action-success`) || '0';
    const errors = await context.redis.get(`metrics:${date}:action-errors`) || '0';

    const stats = [
      `**Today's Metrics**`,
      ``,
      `Clicks: ${clicks}`,
      `Success: ${success}`,
      `Errors: ${errors}`,
    ].join('\n');

    context.ui.showToast(stats);
  },
});
```

## Troubleshooting Production Issues

### 1. Check Logs

```bash
devvit logs <app-name> --since 1h
```

Look for:
- Error messages
- Unexpected behavior
- Performance issues

### 2. Test Locally

Reproduce the issue:
1. Pull latest code
2. Run `devvit dev`
3. Try to replicate the bug
4. Add debug logging
5. Fix and test

### 3. Rollback if Needed

If critical bug:
1. Fix the issue
2. Test thoroughly
3. Deploy fix ASAP

Or revert to previous version:
1. Checkout previous code
2. `devvit upload`
3. `devvit install <subreddit>`

### 4. Communicate

If app is broken:
- Post in subreddit about the issue
- Estimate fix time
- Disable features if needed

## Key Takeaways

1. **Test locally** - Use `devvit dev` and playground
2. **Debug with logging** - Console.log is your friend
3. **Handle errors** - Comprehensive try-catch blocks
4. **Pre-deploy checklist** - Verify everything works
5. **Version properly** - Use semantic versioning
6. **Monitor production** - Watch logs and metrics
7. **Respond quickly** - Fix critical bugs immediately
8. **Communicate** - Keep users informed

## Checkpoint Questions

1. What command starts the local development server?
2. How do you view production logs?
3. What should you do before deploying?
4. How do you update an installed app?
5. What's semantic versioning?

<details>
<summary>Click to see answers</summary>

1. `devvit dev`
2. `devvit logs <app-name>`
3. Run through pre-deployment checklist: test all features, verify configuration, check for errors, etc.
4. Increment version, `devvit build`, `devvit upload`, `devvit install <subreddit>`
5. Version format MAJOR.MINOR.PATCH (e.g., 1.2.3) - increment PATCH for bugs, MINOR for features, MAJOR for breaking changes

</details>

## Final Project Ideas

Apply everything you've learned:

### 1. Community Game Bot
- Custom post type with interactive game
- Leaderboard stored in Redis
- Daily/weekly resets via scheduler
- User stats tracking

### 2. Moderation Assistant
- Auto-flair posts based on content
- Track moderation actions
- Generate weekly mod reports
- Configurable rules via settings

### 3. Engagement Tracker
- Track user contributions
- Award flair for milestones
- Generate community stats
- Scheduled weekly recaps

### 4. Content Scheduler
- Let mods schedule posts
- Store in Redis with scheduled jobs
- Form to create scheduled posts
- List and manage scheduled content

### 5. External Integration
- Fetch data from external API
- Display in custom posts
- Cache results in Redis
- Configurable API keys via settings

## Congratulations!

You've completed the Learn Devvit course! You now have the skills to:

✅ Build Devvit apps from scratch
✅ Work with Reddit's API
✅ Create interactive UIs and forms
✅ Manage persistent state with Redis
✅ Schedule background jobs
✅ Integrate external APIs
✅ Deploy production-ready apps

## Next Steps

1. **Build your own app** - Start with a simple idea
2. **Join the community** - r/Devvit for help and inspiration
3. **Share your work** - Publish to Reddit App Directory
4. **Keep learning** - Devvit is actively developed, stay updated
5. **Contribute** - Help others in the community

## Resources

- [Devvit Documentation](https://developers.reddit.com/docs)
- [r/Devvit Community](https://reddit.com/r/devvit)
- [GitHub Examples](https://github.com/reddit/devvit-examples)
- [Developer Portal](https://developers.reddit.com)

---

**Course complete!** 🎉

Thank you for learning with us. Now go build something amazing!
