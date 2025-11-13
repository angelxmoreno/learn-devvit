# Lesson 5B: Web Views and Advanced UI

**Duration**: 2.5 hours
**Level**: Advanced

> **Note**: This is Part B of a two-part lesson on Devvit UIs:
> - **[Lesson 5A](./05a-forms-and-blocks.md)**: Forms and Blocks UI - inline feed experiences
> - **Lesson 5B** (this lesson): Web Views - full web applications
>
> If you're new to Devvit, start with 5A. Come here when you need more power and flexibility.

## Overview

In this lesson, you'll learn how to build **full web applications** within Devvit using Web Views. Unlike Blocks (which render inline in feeds), Web Views let you create complete HTML/CSS/JavaScript applications that open in a modal or dedicated view.

**What you'll build:**
- Vanilla JavaScript web apps
- React-based applications
- Communication between Devvit and your web app
- Complex interactive experiences

## What Are Web Views?

### The Key Difference

**Blocks (Lesson 5A):**
```
User scrolls feed → Sees your UI inline → Interacts without clicking
```

**Web Views (This lesson):**
```
User clicks post → Modal/view opens → Full web app loads → Rich interaction
```

### When Web Views Shine

✅ **Complex UIs** - Canvas drawing, data visualization, advanced games
✅ **Existing libraries** - Use D3.js, Three.js, Chart.js, etc.
✅ **Custom styling** - Complete CSS control
✅ **Web expertise** - Leverage your existing web dev skills
✅ **Rich interactions** - Drag-and-drop, animations, complex state

### Trade-offs

❌ **Not visible in feed** - Users must click to see
❌ **More complex** - Requires bundling, asset management
❌ **Larger bundle** - Impacts load time
❌ **Mobile optimization** - You handle responsive design

## Project Structure for Web Views

```
my-app/
├── src/
│   └── main.tsx              # Devvit entry point
├── webview/
│   ├── index.html            # Your web app
│   ├── style.css             # Styles
│   ├── app.js                # JavaScript
│   └── assets/               # Images, fonts, etc.
├── devvit.yaml
├── package.json
└── tsconfig.json
```

## Example 1: Vanilla JavaScript Web View

### Step 1: Create the Web App

**`webview/index.html`**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>My Web View</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <div class="container">
    <h1>Hello from Web View!</h1>
    <p>This is a full HTML page.</p>

    <div id="counter-display">Count: <span id="count">0</span></div>

    <button id="increment-btn">Increment</button>
    <button id="fetch-data-btn">Get Reddit Data</button>

    <div id="data-display"></div>
  </div>

  <script src="app.js"></script>
</body>
</html>
```

**`webview/style.css`**
```css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
}

.container {
  background: white;
  border-radius: 12px;
  padding: 32px;
  max-width: 500px;
  width: 100%;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.2);
}

h1 {
  color: #333;
  margin-bottom: 16px;
}

p {
  color: #666;
  margin-bottom: 24px;
}

#counter-display {
  font-size: 24px;
  font-weight: bold;
  color: #667eea;
  margin: 20px 0;
  text-align: center;
}

button {
  background: #667eea;
  color: white;
  border: none;
  padding: 12px 24px;
  border-radius: 6px;
  font-size: 16px;
  cursor: pointer;
  margin: 8px;
  transition: background 0.3s;
}

button:hover {
  background: #5568d3;
}

button:active {
  transform: scale(0.98);
}

#data-display {
  margin-top: 24px;
  padding: 16px;
  background: #f7f7f7;
  border-radius: 8px;
  font-size: 14px;
  color: #333;
  display: none;
}

#data-display.visible {
  display: block;
}
```

**`webview/app.js`**
```javascript
// Communication channel with Devvit
window.addEventListener('message', (event) => {
  const { type, data } = event.data;

  if (type === 'devvit-message') {
    handleDevvitMessage(data);
  }
});

// Send message to Devvit
function sendToDevvit(message) {
  window.parent.postMessage(message, '*');
}

// Counter logic
let count = 0;
const countDisplay = document.getElementById('count');
const incrementBtn = document.getElementById('increment-btn');

incrementBtn.addEventListener('click', () => {
  count++;
  countDisplay.textContent = count;

  // Notify Devvit
  sendToDevvit({
    type: 'counter-updated',
    count: count
  });
});

// Fetch data from Devvit
const fetchDataBtn = document.getElementById('fetch-data-btn');
const dataDisplay = document.getElementById('data-display');

fetchDataBtn.addEventListener('click', () => {
  sendToDevvit({
    type: 'fetch-post-data',
  });
});

function handleDevvitMessage(data) {
  if (data.type === 'post-data') {
    dataDisplay.textContent = `Post: "${data.title}" - ${data.score} points`;
    dataDisplay.classList.add('visible');
  }
}

// Initialize
sendToDevvit({ type: 'webview-ready' });
```

### Step 2: Configure Devvit

**`src/main.tsx`**
```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
  redis: true,
});

// Define custom post with Web View
Devvit.addCustomPostType({
  name: 'web-view-post',
  description: 'A post with a web view',
  height: 'tall',
  render: (context) => {
    // Create communication channel
    const [webviewVisible, setWebviewVisible] = context.useState(false);

    // Handle messages from web view
    const onMessage = async (msg: any) => {
      console.log('Message from webview:', msg);

      switch (msg.type) {
        case 'webview-ready':
          console.log('Web view is ready');
          break;

        case 'counter-updated':
          console.log('Counter updated to:', msg.count);
          // Save to Redis if needed
          await context.redis.set(`counter_${context.postId}`, String(msg.count));
          break;

        case 'fetch-post-data':
          // Fetch Reddit data and send back
          const post = await context.reddit.getPostById(context.postId!);
          context.ui.webView.postMessage('myWebView', {
            type: 'post-data',
            title: post.title,
            score: post.score,
          });
          break;
      }
    };

    // Render Blocks UI with button to open Web View
    return (
      <vstack padding="medium" gap="medium" alignment="center middle">
        <text size="large" weight="bold">
          Interactive Web App
        </text>

        <text color="neutral-content-weak">
          Click below to open the full web experience
        </text>

        <button
          onPress={() => setWebviewVisible(true)}
          appearance="primary"
          size="large"
        >
          Open Web View
        </button>

        <vstack width="100%" height="100%">
          <vstack border="thick" borderColor="neutral-border" cornerRadius="medium">
            <vstack width="100%" height="100%" alignment="center middle">
              {webviewVisible ? (
                <webview
                  id="myWebView"
                  url="webview/index.html"
                  onMessage={onMessage}
                  width="100%"
                  height="100%"
                />
              ) : (
                <text>Web view not loaded</text>
              )}
            </vstack>
          </vstack>
        </vstack>
      </vstack>
    );
  },
});

// Menu action to create the post
Devvit.addMenuItem({
  label: 'Create Web View Post',
  location: 'subreddit',
  onPress: async (event, context) => {
    const subreddit = await context.reddit.getCurrentSubreddit();

    await context.reddit.submitPost({
      title: 'Interactive Web View Post',
      subredditName: subreddit.name,
      preview: (
        <vstack padding="medium" alignment="center middle">
          <text>Click to open interactive web app</text>
        </vstack>
      ),
    });

    context.ui.showToast('Web view post created!');
  },
});

export default Devvit;
```

## Example 2: React Web View

### Step 1: Setup React

**Install dependencies:**
```bash
npm install react react-dom
npm install --save-dev @types/react @types/react-dom
npm install --save-dev vite
```

**`webview/package.json`**
```json
{
  "name": "webview",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "@vitejs/plugin-react": "^4.0.0",
    "vite": "^4.3.0"
  }
}
```

**`webview/vite.config.ts`**
```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  build: {
    outDir: '../public/webview',
    emptyOutDir: true,
  },
});
```

**`webview/src/App.tsx`**
```tsx
import React, { useState, useEffect } from 'react';
import './App.css';

interface DevvitMessage {
  type: string;
  data?: any;
}

function App() {
  const [count, setCount] = useState(0);
  const [postData, setPostData] = useState<any>(null);
  const [username, setUsername] = useState<string>('');

  // Send message to Devvit
  const sendToDevvit = (message: any) => {
    window.parent.postMessage(message, '*');
  };

  // Listen for messages from Devvit
  useEffect(() => {
    const handleMessage = (event: MessageEvent) => {
      const { type, data } = event.data;

      switch (type) {
        case 'devvit-message':
          handleDevvitMessage(data);
          break;
      }
    };

    window.addEventListener('message', handleMessage);

    // Notify Devvit that we're ready
    sendToDevvit({ type: 'webview-ready' });

    return () => window.removeEventListener('message', handleMessage);
  }, []);

  const handleDevvitMessage = (data: any) => {
    switch (data.type) {
      case 'post-data':
        setPostData(data);
        break;
      case 'user-data':
        setUsername(data.username);
        break;
    }
  };

  const handleIncrement = () => {
    const newCount = count + 1;
    setCount(newCount);
    sendToDevvit({ type: 'counter-updated', count: newCount });
  };

  const fetchPostData = () => {
    sendToDevvit({ type: 'fetch-post-data' });
  };

  return (
    <div className="app">
      <div className="card">
        <h1>React Web View</h1>
        <p>Full React app running in Devvit!</p>

        <div className="counter-section">
          <h2>Counter: {count}</h2>
          <button onClick={handleIncrement}>Increment</button>
        </div>

        <div className="data-section">
          <button onClick={fetchPostData}>Fetch Post Data</button>
          {postData && (
            <div className="post-info">
              <h3>{postData.title}</h3>
              <p>Score: {postData.score}</p>
              <p>Comments: {postData.comments}</p>
            </div>
          )}
        </div>

        {username && (
          <div className="user-info">
            <p>Hello, u/{username}!</p>
          </div>
        )}
      </div>
    </div>
  );
}

export default App;
```

**`webview/src/App.css`**
```css
.app {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

.card {
  background: white;
  border-radius: 16px;
  padding: 40px;
  max-width: 600px;
  width: 100%;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
}

h1 {
  color: #333;
  margin-bottom: 8px;
}

p {
  color: #666;
  margin-bottom: 32px;
}

.counter-section {
  background: #f7f7f7;
  padding: 24px;
  border-radius: 12px;
  margin-bottom: 24px;
  text-align: center;
}

.counter-section h2 {
  font-size: 32px;
  color: #667eea;
  margin-bottom: 16px;
}

button {
  background: #667eea;
  color: white;
  border: none;
  padding: 14px 28px;
  border-radius: 8px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s;
}

button:hover {
  background: #5568d3;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

button:active {
  transform: translateY(0);
}

.data-section {
  margin-top: 24px;
}

.post-info {
  background: #f0f4ff;
  padding: 20px;
  border-radius: 8px;
  margin-top: 16px;
  border-left: 4px solid #667eea;
}

.post-info h3 {
  color: #333;
  margin-bottom: 12px;
}

.post-info p {
  color: #666;
  margin: 4px 0;
}

.user-info {
  margin-top: 24px;
  padding: 16px;
  background: #e8f5e9;
  border-radius: 8px;
  text-align: center;
}

.user-info p {
  color: #2e7d32;
  font-weight: 600;
  margin: 0;
}
```

**`webview/src/main.tsx`**
```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './index.css';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

**`webview/index.html`**
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>React Devvit App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

### Step 2: Build Script

Add to your main `package.json`:

```json
{
  "scripts": {
    "build": "npm run build:webview && devvit build",
    "build:webview": "cd webview && npm run build",
    "dev": "devvit dev",
    "dev:webview": "cd webview && npm run dev"
  }
}
```

### Step 3: Devvit Integration

Same as vanilla example, but point to the built files:

```typescript
<webview
  id="myWebView"
  url="public/webview/index.html"  // Built React app
  onMessage={onMessage}
  width="100%"
  height="100%"
/>
```

## Communication: Devvit ↔ Web View

### From Web View to Devvit

**In your web app:**
```javascript
// Send message
window.parent.postMessage({
  type: 'my-action',
  data: { key: 'value' }
}, '*');
```

**In Devvit:**
```typescript
const onMessage = async (msg: any) => {
  if (msg.type === 'my-action') {
    console.log('Received:', msg.data);

    // Access Reddit API
    const post = await context.reddit.getPostById(context.postId!);

    // Access Redis
    await context.redis.set('key', 'value');
  }
};

<webview onMessage={onMessage} ... />
```

### From Devvit to Web View

**In Devvit:**
```typescript
// Send data to web view
context.ui.webView.postMessage('myWebView', {
  type: 'update-data',
  payload: {
    score: post.score,
    comments: post.numberOfComments,
  },
});
```

**In your web app:**
```javascript
window.addEventListener('message', (event) => {
  const { type, payload } = event.data;

  if (type === 'update-data') {
    console.log('Received from Devvit:', payload);
    // Update UI
  }
});
```

## Complete Example: Interactive Game

**`webview/game.html`**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Guess the Number</title>
  <style>
    body {
      margin: 0;
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .game-container {
      background: white;
      padding: 40px;
      border-radius: 20px;
      box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
      max-width: 500px;
      width: 90%;
      text-align: center;
    }
    h1 { color: #1e3c72; margin-bottom: 20px; }
    p { color: #666; margin-bottom: 30px; }
    input {
      width: 100%;
      padding: 15px;
      font-size: 18px;
      border: 2px solid #ddd;
      border-radius: 10px;
      margin-bottom: 15px;
      box-sizing: border-box;
    }
    button {
      width: 100%;
      padding: 15px;
      background: #1e3c72;
      color: white;
      border: none;
      border-radius: 10px;
      font-size: 18px;
      cursor: pointer;
      transition: background 0.3s;
    }
    button:hover { background: #2a5298; }
    .feedback {
      margin-top: 20px;
      padding: 15px;
      border-radius: 10px;
      font-weight: 600;
      display: none;
    }
    .feedback.visible { display: block; }
    .feedback.correct {
      background: #d4edda;
      color: #155724;
    }
    .feedback.wrong {
      background: #f8d7da;
      color: #721c24;
    }
    .stats {
      margin-top: 30px;
      padding: 20px;
      background: #f7f7f7;
      border-radius: 10px;
    }
    .stat-item {
      display: flex;
      justify-content: space-between;
      margin: 10px 0;
      font-size: 16px;
    }
    .leaderboard {
      margin-top: 20px;
      text-align: left;
    }
    .leaderboard-item {
      padding: 10px;
      background: white;
      margin: 5px 0;
      border-radius: 5px;
      display: flex;
      justify-content: space-between;
    }
  </style>
</head>
<body>
  <div class="game-container">
    <h1>🎯 Guess the Number</h1>
    <p>I'm thinking of a number between 1 and 100</p>

    <input
      type="number"
      id="guess-input"
      placeholder="Enter your guess"
      min="1"
      max="100"
    />

    <button id="guess-btn">Make Guess</button>
    <button id="new-game-btn" style="background: #6c757d; margin-top: 10px;">New Game</button>

    <div id="feedback" class="feedback"></div>

    <div class="stats">
      <div class="stat-item">
        <span>Attempts:</span>
        <strong id="attempts">0</strong>
      </div>
      <div class="stat-item">
        <span>Best Score:</span>
        <strong id="best-score">-</strong>
      </div>
      <div class="stat-item">
        <span>Games Played:</span>
        <strong id="games-played">0</strong>
      </div>
    </div>

    <div class="leaderboard">
      <h3>Top Scores</h3>
      <div id="leaderboard-list"></div>
    </div>
  </div>

  <script>
    let targetNumber = Math.floor(Math.random() * 100) + 1;
    let attempts = 0;

    const guessInput = document.getElementById('guess-input');
    const guessBtn = document.getElementById('guess-btn');
    const newGameBtn = document.getElementById('new-game-btn');
    const feedback = document.getElementById('feedback');
    const attemptsDisplay = document.getElementById('attempts');
    const bestScoreDisplay = document.getElementById('best-score');
    const gamesPlayedDisplay = document.getElementById('games-played');
    const leaderboardList = document.getElementById('leaderboard-list');

    // Communication with Devvit
    function sendToDevvit(message) {
      window.parent.postMessage(message, '*');
    }

    window.addEventListener('message', (event) => {
      const { type, data } = event.data;

      if (type === 'devvit-message') {
        handleDevvitMessage(data);
      }
    });

    function handleDevvitMessage(data) {
      if (data.type === 'stats-update') {
        bestScoreDisplay.textContent = data.bestScore || '-';
        gamesPlayedDisplay.textContent = data.gamesPlayed || 0;
      }

      if (data.type === 'leaderboard-update') {
        updateLeaderboard(data.leaderboard);
      }
    }

    function updateLeaderboard(leaderboard) {
      leaderboardList.innerHTML = '';
      leaderboard.slice(0, 5).forEach((entry, index) => {
        const item = document.createElement('div');
        item.className = 'leaderboard-item';
        item.innerHTML = `
          <span>${index + 1}. ${entry.username}</span>
          <strong>${entry.score} attempts</strong>
        `;
        leaderboardList.appendChild(item);
      });
    }

    guessBtn.addEventListener('click', makeGuess);
    guessInput.addEventListener('keypress', (e) => {
      if (e.key === 'Enter') makeGuess();
    });

    function makeGuess() {
      const guess = parseInt(guessInput.value);

      if (!guess || guess < 1 || guess > 100) {
        showFeedback('Please enter a number between 1 and 100', 'wrong');
        return;
      }

      attempts++;
      attemptsDisplay.textContent = attempts;

      if (guess === targetNumber) {
        showFeedback(`🎉 Correct! You won in ${attempts} attempts!`, 'correct');
        guessBtn.disabled = true;

        // Send win to Devvit
        sendToDevvit({
          type: 'game-won',
          attempts: attempts
        });
      } else if (guess < targetNumber) {
        showFeedback('📈 Too low! Try higher', 'wrong');
      } else {
        showFeedback('📉 Too high! Try lower', 'wrong');
      }

      guessInput.value = '';
      guessInput.focus();
    }

    function showFeedback(message, type) {
      feedback.textContent = message;
      feedback.className = `feedback visible ${type}`;
    }

    newGameBtn.addEventListener('click', () => {
      targetNumber = Math.floor(Math.random() * 100) + 1;
      attempts = 0;
      attemptsDisplay.textContent = '0';
      guessInput.value = '';
      guessBtn.disabled = false;
      feedback.classList.remove('visible');

      sendToDevvit({ type: 'new-game' });
    });

    // Initialize
    sendToDevvit({ type: 'webview-ready' });
    sendToDevvit({ type: 'request-stats' });
  </script>
</body>
</html>
```

**`src/main.tsx`** (Devvit side)
```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
  redis: true,
});

interface GameStats {
  gamesPlayed: number;
  bestScore: number;
}

interface LeaderboardEntry {
  username: string;
  score: number;
  timestamp: number;
}

Devvit.addCustomPostType({
  name: 'number-guessing-game',
  description: 'Interactive number guessing game',
  height: 'tall',
  render: (context) => {
    const [webviewVisible, setWebviewVisible] = context.useState(false);

    const onMessage = async (msg: any) => {
      const user = await context.reddit.getCurrentUser();

      switch (msg.type) {
        case 'webview-ready':
          setWebviewVisible(true);
          break;

        case 'request-stats':
          // Get user stats
          const statsKey = `game:stats:${user.id}`;
          const statsStr = await context.redis.get(statsKey);
          const stats: GameStats = statsStr
            ? JSON.parse(statsStr)
            : { gamesPlayed: 0, bestScore: 0 };

          context.ui.webView.postMessage('myWebView', {
            type: 'stats-update',
            ...stats,
          });

          // Get leaderboard
          const leaderboardStr = await context.redis.get('game:leaderboard');
          const leaderboard: LeaderboardEntry[] = leaderboardStr
            ? JSON.parse(leaderboardStr)
            : [];

          context.ui.webView.postMessage('myWebView', {
            type: 'leaderboard-update',
            leaderboard,
          });
          break;

        case 'game-won':
          // Update user stats
          const userStatsKey = `game:stats:${user.id}`;
          const userStatsStr = await context.redis.get(userStatsKey);
          const userStats: GameStats = userStatsStr
            ? JSON.parse(userStatsStr)
            : { gamesPlayed: 0, bestScore: 0 };

          userStats.gamesPlayed++;
          if (userStats.bestScore === 0 || msg.attempts < userStats.bestScore) {
            userStats.bestScore = msg.attempts;
          }

          await context.redis.set(userStatsKey, JSON.stringify(userStats));

          // Update leaderboard
          const lbStr = await context.redis.get('game:leaderboard');
          let leaderboard: LeaderboardEntry[] = lbStr ? JSON.parse(lbStr) : [];

          // Add/update user entry
          const existingIndex = leaderboard.findIndex(e => e.username === user.username);
          const newEntry: LeaderboardEntry = {
            username: user.username,
            score: msg.attempts,
            timestamp: Date.now(),
          };

          if (existingIndex >= 0) {
            if (msg.attempts < leaderboard[existingIndex].score) {
              leaderboard[existingIndex] = newEntry;
            }
          } else {
            leaderboard.push(newEntry);
          }

          // Sort and keep top 10
          leaderboard.sort((a, b) => a.score - b.score);
          leaderboard = leaderboard.slice(0, 10);

          await context.redis.set('game:leaderboard', JSON.stringify(leaderboard));

          // Send updates back
          context.ui.webView.postMessage('myWebView', {
            type: 'stats-update',
            ...userStats,
          });

          context.ui.webView.postMessage('myWebView', {
            type: 'leaderboard-update',
            leaderboard,
          });
          break;

        case 'new-game':
          // Request fresh stats
          const freshStatsKey = `game:stats:${user.id}`;
          const freshStatsStr = await context.redis.get(freshStatsKey);
          const freshStats = freshStatsStr ? JSON.parse(freshStatsStr) : { gamesPlayed: 0, bestScore: 0 };

          context.ui.webView.postMessage('myWebView', {
            type: 'stats-update',
            ...freshStats,
          });
          break;
      }
    };

    return (
      <vstack width="100%" height="100%" alignment="center middle">
        {webviewVisible ? (
          <webview
            id="myWebView"
            url="webview/game.html"
            onMessage={onMessage}
            width="100%"
            height="100%"
          />
        ) : (
          <vstack padding="medium" alignment="center middle" gap="medium">
            <text size="large">Loading game...</text>
          </vstack>
        )}
      </vstack>
    );
  },
});

Devvit.addMenuItem({
  label: 'Create Guessing Game',
  location: 'subreddit',
  onPress: async (event, context) => {
    const subreddit = await context.reddit.getCurrentSubreddit();

    await context.reddit.submitPost({
      title: '🎯 Guess the Number Game',
      subredditName: subreddit.name,
      preview: (
        <vstack padding="medium" alignment="center middle" gap="small">
          <text size="large" weight="bold">🎯 Guess the Number</text>
          <text>Click to play!</text>
        </vstack>
      ),
    });

    context.ui.showToast('Game created!');
  },
});

export default Devvit;
```

## Best Practices

### 1. Responsive Design

Always support mobile:

```css
/* Mobile-first approach */
.container {
  width: 100%;
  padding: 20px;
}

@media (min-width: 768px) {
  .container {
    max-width: 600px;
    padding: 40px;
  }
}
```

### 2. Loading States

Show loading while web view initializes:

```typescript
render: (context) => {
  const [isReady, setIsReady] = context.useState(false);

  const onMessage = async (msg: any) => {
    if (msg.type === 'webview-ready') {
      setIsReady(true);
    }
  };

  return (
    <vstack width="100%" height="100%">
      {!isReady && (
        <vstack alignment="center middle">
          <text>Loading...</text>
        </vstack>
      )}
      <webview
        url="webview/index.html"
        onMessage={onMessage}
        width="100%"
        height="100%"
      />
    </vstack>
  );
}
```

### 3. Error Handling

Catch web view errors:

```javascript
window.addEventListener('error', (event) => {
  console.error('Web view error:', event.error);

  sendToDevvit({
    type: 'error',
    message: event.error.message,
  });
});
```

### 4. Asset Management

Keep assets small and optimized:
- Compress images
- Minify CSS/JS
- Use CDNs for libraries when possible
- Lazy load heavy resources

### 5. Message Type Safety

Use TypeScript for message contracts:

```typescript
// shared/types.ts
export type WebViewMessage =
  | { type: 'webview-ready' }
  | { type: 'fetch-data'; id: string }
  | { type: 'update-count'; count: number };

export type DevvitMessage =
  | { type: 'data-response'; data: any }
  | { type: 'error'; message: string };
```

## Blocks vs Web Views: Decision Matrix

| Use Case | Recommended Approach |
|----------|---------------------|
| Simple poll/voting | ✅ Blocks |
| Data dashboard | ⚠️ Blocks (if simple) or Web Views (if complex) |
| Text-based game | ✅ Blocks |
| Canvas-based game | ✅ Web Views |
| Form with validation | ✅ Blocks (use Devvit forms) |
| Rich text editor | ✅ Web Views |
| Leaderboard display | ✅ Blocks |
| Interactive chart (D3.js) | ✅ Web Views |
| Button grid | ✅ Blocks |
| Drag-and-drop interface | ✅ Web Views |
| Timer/countdown | ⚠️ Either (Blocks simpler) |
| Video player | ✅ Web Views |
| Image gallery | ⚠️ Either (depends on interactivity) |
| Embedded map | ✅ Web Views |

## Common Pitfalls

### 1. Bundle Size

❌ **Don't:** Include entire libraries for small features
```javascript
import _ from 'lodash'; // 70KB!
```

✅ **Do:** Import only what you need
```javascript
import debounce from 'lodash/debounce'; // 2KB
```

### 2. State Synchronization

❌ **Don't:** Keep critical state only in web view
```javascript
// Data lost on web view reload!
let score = 100;
```

✅ **Do:** Sync state to Devvit/Redis
```javascript
// Send to Devvit for persistence
sendToDevvit({ type: 'save-score', score: 100 });
```

### 3. CORS Issues

❌ **Don't:** Try to fetch external resources directly
```javascript
fetch('https://api.example.com/data'); // May fail
```

✅ **Do:** Proxy through Devvit
```javascript
// Ask Devvit to fetch (it has http: true configured)
sendToDevvit({ type: 'fetch-external-data' });
```

## Key Takeaways

1. **Web Views = Full Web Apps** - HTML, CSS, JS with unlimited flexibility
2. **Not visible in feed** - Users must click to see
3. **Use for complex UIs** - Games, visualizations, advanced interactions
4. **Message passing** - Communication via `postMessage`
5. **Build step required** - Bundle your web app before deployment
6. **Mobile-first** - Always design for small screens
7. **State in Redis** - Don't rely on web view memory
8. **Choose wisely** - Start with Blocks, upgrade to Web Views when needed

## Checkpoint Questions

1. What's the main difference between Blocks and Web Views?
2. How do you send a message from the web view to Devvit?
3. How do you send a message from Devvit to the web view?
4. Why should you store important state in Redis, not just in the web view?
5. When should you use Web Views instead of Blocks?

<details>
<summary>Click to see answers</summary>

1. Blocks render inline in feeds, Web Views open in a modal/dedicated view
2. `window.parent.postMessage({ type: 'my-message', data: {...} }, '*')`
3. `context.ui.webView.postMessage('webViewId', { type: 'message', data: {...} })`
4. Web views can reload/unmount, losing all JavaScript state. Redis persists data between sessions.
5. When you need complex UI (canvas, charts), custom CSS, existing web libraries, or interactions that Blocks can't provide

</details>

## Practice Exercises

### Exercise 1: Todo List

Create a web view with a todo list that persists to Redis:
- Add/remove todos
- Mark as complete
- Filter (all/active/completed)
- Sync with Redis

### Exercise 2: Drawing Canvas

Build a simple drawing app:
- HTML5 canvas
- Color picker
- Clear button
- Save drawings to Reddit as images

### Exercise 3: Quiz Game

Multi-question quiz with:
- Timer per question
- Score tracking
- Leaderboard (Redis)
- Share results to Reddit

## Additional Resources

- [Web Views Documentation](https://developers.reddit.com/docs/web-views)
- [postMessage API](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)
- [Vite Documentation](https://vitejs.dev/)
- [React Documentation](https://react.dev/)

---

➡️ **Continue to [Lesson 6: Reddit API Integration](./06-reddit-api.md)**

**Estimated time to complete**: 2.5 hours
**Practice exercises**: 3 hands-on challenges
**Prerequisites**: Lessons 1-5A completed
