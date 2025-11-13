# Lesson 5A: Working with Forms and Blocks UI

**Duration**: 2 hours
**Level**: Intermediate

> **Note**: This is Part A of a two-part lesson on Devvit UIs:
> - **Lesson 5A** (this lesson): Forms and Blocks UI - inline feed experiences
> - **[Lesson 5B](./05b-web-views.md)**: Web Views - full web applications
>
> Learn about both approaches to choose the right one for your project!

## Overview

In this lesson, you'll learn how to create interactive user interfaces using Devvit's **Blocks** approach. You'll master forms for user input and build custom post types that render directly in Reddit feeds.

Blocks use a React-like syntax with Devvit's component library, perfect for experiences that should appear inline as users scroll.

## Forms: Collecting User Input

Forms are Devvit's way of collecting structured data from users. They're perfect for settings, configurations, and user submissions.

### Basic Form Structure

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
});

// Define a form
const myForm = Devvit.createForm(
  {
    fields: [
      {
        name: 'username',
        label: 'Username',
        type: 'string',
        required: true,
      },
    ],
    title: 'Enter Information',
    acceptLabel: 'Submit',
    cancelLabel: 'Cancel',
  },
  async (event, context) => {
    // Handle form submission
    const username = event.values.username;
    context.ui.showToast(`Hello, ${username}!`);
  }
);

// Show the form from a menu action
Devvit.addMenuItem({
  label: 'Open Form',
  location: 'subreddit',
  onPress: async (event, context) => {
    context.ui.showForm(myForm);
  },
});

export default Devvit;
```

### Form Field Types

#### 1. String Field

For text input:

```typescript
{
  name: 'title',
  label: 'Post Title',
  type: 'string',
  required: true,
  defaultValue: 'Default title',
  helpText: 'Enter a descriptive title',
}
```

#### 2. Number Field

For numeric input:

```typescript
{
  name: 'threshold',
  label: 'Score Threshold',
  type: 'number',
  required: true,
  defaultValue: 100,
  helpText: 'Minimum score required',
}
```

#### 3. Boolean Field

For yes/no choices:

```typescript
{
  name: 'enabled',
  label: 'Enable Feature',
  type: 'boolean',
  required: false,
  defaultValue: true,
  helpText: 'Turn this feature on or off',
}
```

#### 4. Paragraph Field

For multi-line text:

```typescript
{
  name: 'description',
  label: 'Description',
  type: 'paragraph',
  required: false,
  defaultValue: '',
  helpText: 'Enter a detailed description',
  lineHeight: 5,  // Number of visible lines
}
```

#### 5. Select Field

For dropdown choices:

```typescript
{
  name: 'category',
  label: 'Category',
  type: 'select',
  required: true,
  options: [
    { label: 'General', value: 'general' },
    { label: 'Technical', value: 'tech' },
    { label: 'Social', value: 'social' },
  ],
  defaultValue: ['general'],
  multiSelect: false,
}
```

**Multi-select example:**
```typescript
{
  name: 'tags',
  label: 'Tags',
  type: 'select',
  required: false,
  options: [
    { label: 'Urgent', value: 'urgent' },
    { label: 'Bug', value: 'bug' },
    { label: 'Feature', value: 'feature' },
  ],
  multiSelect: true,
  defaultValue: [],
}
```

### Complete Form Example

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
  redis: true,
});

const pollForm = Devvit.createForm(
  {
    title: 'Create a Poll',
    description: 'Create a new poll for your community',
    acceptLabel: 'Create Poll',
    fields: [
      {
        name: 'question',
        label: 'Poll Question',
        type: 'string',
        required: true,
        helpText: 'What do you want to ask?',
      },
      {
        name: 'options',
        label: 'Options (comma-separated)',
        type: 'paragraph',
        required: true,
        helpText: 'Enter options separated by commas',
        lineHeight: 3,
      },
      {
        name: 'duration',
        label: 'Duration (hours)',
        type: 'number',
        required: true,
        defaultValue: 24,
        helpText: 'How long should the poll run?',
      },
      {
        name: 'allowMultiple',
        label: 'Allow Multiple Votes',
        type: 'boolean',
        defaultValue: false,
        helpText: 'Can users select multiple options?',
      },
    ],
  },
  async (event, context) => {
    try {
      const { question, options, duration, allowMultiple } = event.values;

      // Parse options
      const optionList = (options as string)
        .split(',')
        .map(o => o.trim())
        .filter(o => o.length > 0);

      if (optionList.length < 2) {
        context.ui.showToast({
          text: 'Please provide at least 2 options',
          appearance: 'error',
        });
        return;
      }

      // Store poll data
      const pollId = `poll_${Date.now()}`;
      await context.redis.set(
        pollId,
        JSON.stringify({
          question,
          options: optionList,
          duration,
          allowMultiple,
          votes: {},
          createdAt: Date.now(),
        })
      );

      context.ui.showToast({
        text: `Poll created: ${question}`,
        appearance: 'success',
      });
    } catch (error) {
      console.error('Error creating poll:', error);
      context.ui.showToast({
        text: 'Failed to create poll',
        appearance: 'error',
      });
    }
  }
);

Devvit.addMenuItem({
  label: 'Create Poll',
  location: 'subreddit',
  forUserType: 'moderator',
  onPress: async (event, context) => {
    context.ui.showForm(pollForm);
  },
});

export default Devvit;
```

### Form Validation

Handle validation in the submission handler:

```typescript
const registrationForm = Devvit.createForm(
  {
    title: 'Register',
    fields: [
      { name: 'email', label: 'Email', type: 'string', required: true },
      { name: 'age', label: 'Age', type: 'number', required: true },
    ],
  },
  async (event, context) => {
    const { email, age } = event.values;

    // Validate email
    if (!email.includes('@')) {
      context.ui.showToast({
        text: 'Invalid email address',
        appearance: 'error',
      });
      return;
    }

    // Validate age
    if (age < 13) {
      context.ui.showToast({
        text: 'You must be at least 13 years old',
        appearance: 'error',
      });
      return;
    }

    // Process registration
    await context.redis.set(`user_${email}`, JSON.stringify({ email, age }));

    context.ui.showToast({
      text: 'Registration successful!',
      appearance: 'success',
    });
  }
);
```

### Dynamic Forms

You can populate form fields dynamically:

```typescript
Devvit.addMenuItem({
  label: 'Edit Settings',
  location: 'subreddit',
  onPress: async (event, context) => {
    // Fetch current settings
    const currentTitle = await context.redis.get('title') || 'Default Title';
    const currentThreshold = await context.redis.get('threshold') || '100';

    // Create form with current values
    const settingsForm = Devvit.createForm(
      {
        title: 'Edit Settings',
        fields: [
          {
            name: 'title',
            label: 'Title',
            type: 'string',
            defaultValue: currentTitle,
          },
          {
            name: 'threshold',
            label: 'Threshold',
            type: 'number',
            defaultValue: Number(currentThreshold),
          },
        ],
      },
      async (event, context) => {
        await context.redis.set('title', event.values.title as string);
        await context.redis.set('threshold', String(event.values.threshold));
        context.ui.showToast('Settings updated!');
      }
    );

    context.ui.showForm(settingsForm);
  },
});
```

## Custom Post Types

Custom post types let you create entirely new kinds of Reddit posts with custom UIs. They use a React-like syntax called **Blocks**.

### Basic Custom Post

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
  redis: true,
});

// Define custom post type
Devvit.addCustomPostType({
  name: 'Hello Post',
  description: 'A simple custom post',
  height: 'regular',
  render: (context) => {
    return (
      <vstack padding="medium" alignment="center middle">
        <text size="xxlarge" weight="bold">
          Hello, Devvit!
        </text>
        <text size="medium" color="neutral-content-weak">
          This is a custom post type
        </text>
      </vstack>
    );
  },
});

// Create menu action to post it
Devvit.addMenuItem({
  label: 'Create Hello Post',
  location: 'subreddit',
  onPress: async (event, context) => {
    const subreddit = await context.reddit.getCurrentSubreddit();

    await context.reddit.submitPost({
      title: 'Hello Post',
      subredditName: subreddit.name,
      preview: (
        <vstack padding="medium" alignment="center middle">
          <text size="large">Click to view</text>
        </vstack>
      ),
    });

    context.ui.showToast('Custom post created!');
  },
});

export default Devvit;
```

### Block Components

Devvit provides several UI building blocks:

#### Layout Components

**`vstack`** - Vertical stack (column):
```typescript
<vstack padding="medium" gap="small" alignment="start top">
  <text>First item</text>
  <text>Second item</text>
</vstack>
```

**`hstack`** - Horizontal stack (row):
```typescript
<hstack padding="medium" gap="small" alignment="start middle">
  <text>Left</text>
  <text>Right</text>
</hstack>
```

**`zstack`** - Layered stack:
```typescript
<zstack width="100%" height="100%">
  <image url="background.jpg" imageWidth={100} imageHeight={100} />
  <text>Overlay text</text>
</zstack>
```

**`spacer`** - Empty space:
```typescript
<vstack>
  <text>Top</text>
  <spacer size="large" />
  <text>Bottom</text>
</vstack>
```

#### Content Components

**`text`** - Display text:
```typescript
<text
  size="large"          // 'xsmall' | 'small' | 'medium' | 'large' | 'xlarge' | 'xxlarge'
  weight="bold"         // 'regular' | 'bold'
  color="primary"       // Various color options
  alignment="center"    // 'start' | 'center' | 'end'
>
  Hello World
</text>
```

**`button`** - Interactive button:
```typescript
<button
  onPress={() => console.log('Clicked!')}
  appearance="primary"    // 'primary' | 'secondary' | 'bordered' | 'plain'
  size="medium"          // 'small' | 'medium' | 'large'
  icon="add"
>
  Click Me
</button>
```

**`image`** - Display images:
```typescript
<image
  url="https://example.com/image.jpg"
  imageWidth={400}
  imageHeight={300}
  description="Alt text"
  resizeMode="cover"    // 'cover' | 'contain' | 'fill'
/>
```

**`icon`** - Built-in icons:
```typescript
<icon
  name="upvote"
  size="small"
  color="upvote"
/>
```

### Interactive Custom Post

```typescript
import { Devvit, useState } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
  redis: true,
});

Devvit.addCustomPostType({
  name: 'Counter Post',
  description: 'A post with an interactive counter',
  height: 'regular',
  render: (context) => {
    // useState hook for local state
    const [count, setCount] = useState(0);

    return (
      <vstack padding="medium" alignment="center middle" gap="medium">
        <text size="xxlarge" weight="bold">
          {count}
        </text>

        <hstack gap="small">
          <button
            onPress={() => setCount(count - 1)}
            appearance="secondary"
          >
            Decrease
          </button>

          <button
            onPress={() => setCount(count + 1)}
            appearance="primary"
          >
            Increase
          </button>
        </hstack>

        <button
          onPress={() => setCount(0)}
          appearance="plain"
          size="small"
        >
          Reset
        </button>
      </vstack>
    );
  },
});

export default Devvit;
```

### useState Hook

Similar to React's useState:

```typescript
import { useState } from '@devvit/public-api';

const [value, setValue] = useState(initialValue);

// Update value
setValue(newValue);

// Or use updater function
setValue(prev => prev + 1);
```

### Persistent State with Redis

Combine useState with Redis for persistent data:

```typescript
import { Devvit, useState, useAsync } from '@devvit/public-api';

Devvit.addCustomPostType({
  name: 'Persistent Counter',
  height: 'regular',
  render: (context) => {
    const [count, setCount] = useState(async () => {
      // Load initial value from Redis
      const saved = await context.redis.get('global-count');
      return Number(saved || 0);
    });

    const increment = async () => {
      const newCount = count + 1;
      setCount(newCount);
      await context.redis.set('global-count', String(newCount));
    };

    return (
      <vstack padding="medium" alignment="center middle" gap="medium">
        <text size="xxlarge">{count}</text>
        <button onPress={increment}>Increment</button>
      </vstack>
    );
  },
});
```

### useAsync Hook

For async operations:

```typescript
import { useAsync } from '@devvit/public-api';

const { data, loading, error } = useAsync(async () => {
  const post = await context.reddit.getPostById(context.postId!);
  return post;
});

if (loading) {
  return <text>Loading...</text>;
}

if (error) {
  return <text>Error: {error.message}</text>;
}

return <text>{data.title}</text>;
```

### Complete Interactive Example: Simple Poll

```typescript
import { Devvit, useState } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
  redis: true,
});

interface PollData {
  question: string;
  options: string[];
  votes: { [key: string]: number };
}

Devvit.addCustomPostType({
  name: 'poll',
  description: 'Interactive poll',
  height: 'tall',
  render: (context) => {
    const [pollData] = useState(async () => {
      const data = await context.redis.get(`poll_${context.postId}`);
      return JSON.parse(data!) as PollData;
    });

    const [votes, setVotes] = useState(pollData.votes);
    const [hasVoted, setHasVoted] = useState(false);

    const vote = async (option: string) => {
      if (hasVoted) {
        context.ui.showToast('You already voted!');
        return;
      }

      const newVotes = { ...votes, [option]: (votes[option] || 0) + 1 };
      setVotes(newVotes);
      setHasVoted(true);

      // Persist to Redis
      await context.redis.set(
        `poll_${context.postId}`,
        JSON.stringify({ ...pollData, votes: newVotes })
      );

      context.ui.showToast('Vote recorded!');
    };

    const totalVotes = Object.values(votes).reduce((a, b) => a + b, 0);

    return (
      <vstack padding="medium" gap="medium">
        <text size="xlarge" weight="bold">
          {pollData.question}
        </text>

        {pollData.options.map(option => {
          const optionVotes = votes[option] || 0;
          const percentage = totalVotes > 0
            ? ((optionVotes / totalVotes) * 100).toFixed(1)
            : '0';

          return (
            <vstack key={option} gap="small" width="100%">
              <button
                onPress={() => vote(option)}
                appearance={hasVoted ? 'bordered' : 'primary'}
                disabled={hasVoted}
                grow
              >
                {option}
              </button>
              {hasVoted && (
                <text size="small" color="neutral-content-weak">
                  {optionVotes} votes ({percentage}%)
                </text>
              )}
            </vstack>
          );
        })}

        <text size="small" color="neutral-content-weak">
          Total votes: {totalVotes}
        </text>
      </vstack>
    );
  },
});

// Menu action to create poll
const pollForm = Devvit.createForm(
  {
    title: 'Create Poll',
    fields: [
      {
        name: 'question',
        label: 'Question',
        type: 'string',
        required: true,
      },
      {
        name: 'options',
        label: 'Options (comma-separated)',
        type: 'paragraph',
        required: true,
      },
    ],
  },
  async (event, context) => {
    const { question, options } = event.values;
    const optionList = (options as string).split(',').map(o => o.trim());

    const subreddit = await context.reddit.getCurrentSubreddit();
    const post = await context.reddit.submitPost({
      title: question as string,
      subredditName: subreddit.name,
      preview: (
        <vstack padding="medium">
          <text>Click to vote in this poll</text>
        </vstack>
      ),
    });

    // Initialize poll data
    await context.redis.set(
      `poll_${post.id}`,
      JSON.stringify({
        question,
        options: optionList,
        votes: {},
      })
    );

    context.ui.showToast('Poll created!');
  }
);

Devvit.addMenuItem({
  label: 'Create Poll',
  location: 'subreddit',
  onPress: (event, context) => context.ui.showForm(pollForm),
});

export default Devvit;
```

### Styling and Layout

#### Alignment

```typescript
// Horizontal alignment
<vstack alignment="start">   // left
<vstack alignment="center">  // center
<vstack alignment="end">     // right

// Vertical + Horizontal
<vstack alignment="center middle">  // centered both ways
<vstack alignment="start top">      // top-left
<vstack alignment="end bottom">     // bottom-right
```

#### Spacing

```typescript
// Padding
<vstack padding="small">     // All sides
<vstack padding="medium">
<vstack padding="large">

// Gap between children
<vstack gap="small">
<vstack gap="medium">
<vstack gap="large">
```

#### Sizing

```typescript
// Fixed size
<vstack width="100px" height="100px">

// Percentage
<vstack width="100%" height="50%">

// Grow to fill
<vstack grow>
```

### Post Height Options

```typescript
Devvit.addCustomPostType({
  name: 'my-post',
  height: 'regular',  // or 'tall'
  // ...
});
```

- **`regular`**: ~300px height
- **`tall`**: ~500px height

## Best Practices

### 1. Form Validation

Always validate user input:

```typescript
const form = Devvit.createForm(/* ... */, async (event, context) => {
  const { username } = event.values;

  if (!username || username.length < 3) {
    context.ui.showToast({
      text: 'Username must be at least 3 characters',
      appearance: 'error',
    });
    return;
  }

  // Process valid input
});
```

### 2. Error Handling in Custom Posts

```typescript
render: (context) => {
  try {
    const [data] = useState(async () => {
      const result = await context.redis.get('key');
      return JSON.parse(result!);
    });

    return <text>{data.message}</text>;
  } catch (error) {
    return (
      <vstack padding="medium" alignment="center middle">
        <text color="error">Failed to load data</text>
      </vstack>
    );
  }
}
```

### 3. Loading States

```typescript
render: (context) => {
  const { data, loading, error } = useAsync(async () => {
    return await fetchData();
  });

  if (loading) {
    return (
      <vstack padding="medium" alignment="center middle">
        <text>Loading...</text>
      </vstack>
    );
  }

  if (error) {
    return <text color="error">Error: {error.message}</text>;
  }

  return <text>{data}</text>;
}
```

### 4. Optimize Re-renders

Don't call expensive operations on every render:

```typescript
// Bad: Fetches on every render
render: (context) => {
  const data = await context.redis.get('key');  // Don't do this!
  return <text>{data}</text>;
}

// Good: Use useState or useAsync
render: (context) => {
  const [data] = useState(async () => {
    return await context.redis.get('key');
  });
  return <text>{data}</text>;
}
```

## Practice Exercises

### Exercise 1: Feedback Form

Create a form that collects user feedback with: name, email, rating (1-5), and comments. Store in Redis.

<details>
<summary>Solution</summary>

```typescript
const feedbackForm = Devvit.createForm(
  {
    title: 'Submit Feedback',
    fields: [
      { name: 'name', label: 'Name', type: 'string', required: true },
      { name: 'email', label: 'Email', type: 'string', required: true },
      {
        name: 'rating',
        label: 'Rating',
        type: 'select',
        options: [
          { label: '1 - Poor', value: '1' },
          { label: '2 - Fair', value: '2' },
          { label: '3 - Good', value: '3' },
          { label: '4 - Very Good', value: '4' },
          { label: '5 - Excellent', value: '5' },
        ],
        required: true,
      },
      {
        name: 'comments',
        label: 'Comments',
        type: 'paragraph',
        lineHeight: 4,
      },
    ],
  },
  async (event, context) => {
    const feedbackId = `feedback_${Date.now()}`;
    await context.redis.set(feedbackId, JSON.stringify(event.values));
    context.ui.showToast('Thank you for your feedback!');
  }
);

Devvit.addMenuItem({
  label: 'Submit Feedback',
  location: 'subreddit',
  onPress: (event, context) => context.ui.showForm(feedbackForm),
});
```
</details>

### Exercise 2: Like Button Post

Create a custom post with a like button that shows the total number of likes (stored in Redis).

<details>
<summary>Solution</summary>

```typescript
Devvit.addCustomPostType({
  name: 'like-post',
  height: 'regular',
  render: (context) => {
    const [likes, setLikes] = useState(async () => {
      const count = await context.redis.get(`likes_${context.postId}`);
      return Number(count || 0);
    });

    const [hasLiked, setHasLiked] = useState(false);

    const handleLike = async () => {
      if (hasLiked) return;

      const newCount = likes + 1;
      setLikes(newCount);
      setHasLiked(true);
      await context.redis.set(`likes_${context.postId}`, String(newCount));
    };

    return (
      <vstack padding="medium" alignment="center middle" gap="medium">
        <text size="xxlarge">{likes}</text>
        <button
          onPress={handleLike}
          appearance={hasLiked ? 'bordered' : 'primary'}
          disabled={hasLiked}
          icon="upvote"
        >
          {hasLiked ? 'Liked!' : 'Like'}
        </button>
      </vstack>
    );
  },
});
```
</details>

## Key Takeaways

1. **Forms collect structured input** - Use for settings, user submissions
2. **Multiple field types** - string, number, boolean, paragraph, select
3. **Validate input** - Always check user data before processing
4. **Custom posts use Blocks** - React-like component syntax
5. **useState for local state** - Reactive UI updates
6. **useAsync for data loading** - Handle async operations properly
7. **Redis for persistence** - Store data between sessions
8. **Loading states** - Always handle loading and errors

## Checkpoint Questions

1. How do you show a form to a user?
2. What's the difference between `string` and `paragraph` field types?
3. What hook do you use for reactive state in custom posts?
4. What are the three main layout components?
5. How do you make state persist between page loads?

<details>
<summary>Click to see answers</summary>

1. `context.ui.showForm(formName)`
2. `string` is single-line input, `paragraph` is multi-line text area
3. `useState` (imported from `@devvit/public-api`)
4. `vstack` (vertical), `hstack` (horizontal), `zstack` (layered)
5. Store state in Redis using `context.redis.set()` and load it with `useState(async () => ...)`

</details>

## Blocks vs. Web Views: Which Should You Use?

**Use Blocks (this lesson) when:**
- ✅ Users should see content while scrolling the feed
- ✅ Simple interactions (voting, buttons, basic displays)
- ✅ You want the fastest development time
- ✅ Mobile-first experience is critical
- ✅ Devvit's components meet your needs

**Use Web Views (next lesson) when:**
- ✅ You need complex UI (canvas, charts, advanced games)
- ✅ You want to use existing web libraries (React, D3.js, etc.)
- ✅ You need precise styling control with custom CSS
- ✅ The experience benefits from opening in a focused view

**Not sure?** Start with Blocks - they're simpler and cover 80% of use cases. You can always upgrade to Web Views later.

## Additional Resources

- [Forms Documentation](https://developers.reddit.com/docs/forms)
- [Custom Posts Documentation](https://developers.reddit.com/docs/custom-posts)
- [Blocks Reference](https://developers.reddit.com/docs/blocks)

---

➡️ **Continue to [Lesson 5B: Web Views and Advanced UI](./05b-web-views.md)** to learn about building full web applications, or skip to [Lesson 6: Reddit API Integration](./06-reddit-api.md)

**Estimated time to complete**: 2 hours
**Practice exercises**: 2 hands-on challenges
**Prerequisites**: Lessons 1-4 completed
