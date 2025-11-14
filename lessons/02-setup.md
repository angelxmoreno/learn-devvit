# Lesson 2: Setting Up Development Environment

**Duration**: 45 minutes
**Level**: Beginner

## Overview

In this lesson, you'll install all the necessary tools and set up your development environment for Devvit development. By the end, you'll have a working Devvit project ready for coding.

## Prerequisites

Before you begin, ensure you have:

- **Reddit Account** - You'll need a Reddit account to develop and test
- **Modern OS** - macOS, Linux, or Windows 10/11
- **Terminal Access** - Command line familiarity
- **Text Editor/IDE** - VS Code recommended

## Step 1: Install Node.js

Devvit requires **Node.js 18.x or higher**.

### Check if Node.js is installed:

```bash
node --version
```

If you see `v18.x.x` or higher, you're good to go. Otherwise, install Node.js:

### Installation Options:

**Option A: Official Installer (Recommended for beginners)**
1. Visit [nodejs.org](https://nodejs.org)
2. Download the LTS version (Long Term Support)
3. Run the installer
4. Verify: `node --version`

**Option B: Node Version Manager (Recommended for developers)**

Using `nvm` lets you manage multiple Node versions:

**macOS/Linux:**
```bash
# Install nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# Reload shell configuration
source ~/.bashrc  # or ~/.zshrc

# Install Node 18
nvm install 18
nvm use 18
```

**Windows:**
1. Download [nvm-windows](https://github.com/coreybutler/nvm-windows/releases)
2. Install and restart terminal
3. Run:
```bash
nvm install 18
nvm use 18
```

### Verify Installation:

```bash
node --version   # Should show v18.x.x or higher
npm --version    # Should show 9.x.x or higher
```

## Step 2: Understanding npx (No Installation Needed!)

Good news: **You don't need to install the Devvit CLI globally!**

Devvit uses `npx`, which comes bundled with Node.js. When you run `npx devvit init`, it automatically downloads and runs the latest version of Devvit without requiring a global installation.

### What is npx?

`npx` is a package runner tool that comes with npm (5.2+). It:
- Downloads and runs packages on-demand
- Always uses the latest version
- Doesn't clutter your global npm packages
- Avoids permission issues with global installs

### Verify npx is available:

```bash
npx --version
```

You should see a version number (e.g., `8.x.x` or higher). If you have Node.js 18+, you already have npx!

### Optional: Install Devvit CLI Globally (Not Recommended)

While not necessary, you *can* install devvit globally if you prefer typing `devvit` instead of `npx devvit`:

```bash
npm install -g devvit
```

However, using `npx devvit` is the recommended approach as it:
- Always uses the latest version
- Avoids permission issues
- Doesn't require maintenance

## Step 3: Understanding Authentication

Unlike many development platforms, Devvit handles authentication through its web-based project creation wizard (which we'll use in Step 5). You don't need to run a separate `devvit login` command before getting started.

### How Authentication Works:

When you visit `https://developers.reddit.com/new` (covered in Step 5), the wizard will:
1. Ask you to log in to Reddit (or use your existing session)
2. Authorize the Devvit platform
3. Generate a unique authentication token
4. Save the token to `~/.devvit/token`
5. Create your project all in one flow

### Alternative: Manual Login (Optional)

If you're already authenticated and want to verify your login status:

```bash
devvit whoami
```

This shows your currently authenticated Reddit username.

To manually log in (typically not needed for first-time setup):
```bash
devvit login
```

**Note:** For most users, authentication happens automatically through the web wizard in Step 5, so you can skip manual login.

## Step 4: Set Up Your IDE

While any text editor works, **Visual Studio Code** is recommended for the best Devvit development experience.

### Install VS Code:

1. Download from [code.visualstudio.com](https://code.visualstudio.com)
2. Install for your operating system

### Recommended Extensions:

Install these VS Code extensions for optimal TypeScript development:

1. **ESLint** (`dbaeumer.vscode-eslint`)
   - Real-time linting for code quality

2. **Prettier** (`esbenp.prettier-vscode`)
   - Automatic code formatting

3. **TypeScript Nightly** (`ms-vscode.vscode-typescript-next`)
   - Latest TypeScript features

4. **Error Lens** (`usernamehw.errorlens`)
   - Inline error highlighting

5. **Path Intellisense** (`christian-kohler.path-intellisense`)
   - Autocomplete file paths

### Install extensions via command line:

```bash
code --install-extension dbaeumer.vscode-eslint
code --install-extension esbenp.prettier-vscode
code --install-extension ms-vscode.vscode-typescript-next
code --install-extension usernamehw.errorlens
code --install-extension christian-kohler.path-intellisense
```

### Configure VS Code Settings:

Create or update `.vscode/settings.json` in your workspace:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true
}
```

## Step 5: Create Your First Project Using the Web Wizard

Devvit uses a web-based wizard to help you create your first project. This wizard will authenticate you and generate the necessary commands.

### Start the project creation wizard:

1. **Open your browser** and visit: `https://developers.reddit.com/new`

2. **Choose a template:**
   - For this lesson, select "Empty" or "Hello World"
   - Other available templates: React, menu-action, scheduled-post, custom-post, etc.

3. **Go through the wizard:**
   - Create or connect your Reddit account (if not already logged in)
   - Authorize Reddit Developers to access your account
   - The wizard will generate custom commands for you

4. **Follow the terminal instructions:**
   - The wizard will display commands to copy and paste into your terminal
   - These commands will authenticate you and create your project
   - Example commands you might see:
   ```bash
   npx devvit init <your-unique-code>
   cd my-first-app
   npm install
   ```

5. **On success**, you should see output like:
   ```
   Your Devvit authentication token has been saved to ~/.devvit/token
   Fetching and extracting the template...
   Cutting the template to the target directory...
   🔧 Installing dependencies...
   🚀🚀🚀 Devvit app successfully initialized!
   ┌────────────────────────────────────────────────────┐
   │ • `cd my-app` to open your project directory       │
   │ • `npm run dev` to develop in your test community  │
   └────────────────────────────────────────────────────┘
   ```

**Important Notes:**
- Each wizard session generates a unique authentication code
- The code is one-time use and expires quickly
- If the wizard times out, start over at `https://developers.reddit.com/new`
- The wizard handles both authentication AND project creation

## Step 6: Verify Project Structure

After the wizard completes, let's verify what was created:

### Project structure:

The wizard creates a standard Devvit project with this structure:

```
my-first-app/
├── src/
│   └── main.tsx          # Your app's entry point
├── devvit.yaml           # App configuration
├── package.json          # Dependencies
├── tsconfig.json         # TypeScript config
└── .gitignore           # Git ignore rules
```

The wizard already runs `npm install` for you, so dependencies are ready to use.

## Step 7: Understand the Project Structure

Let's explore what was created:

### `devvit.yaml` - App Configuration

```yaml
name: my-first-app
version: 0.0.1
author: your-username
description: My first Devvit app
```

This file defines your app's metadata and configuration.

### `package.json` - Dependencies

```json
{
  "name": "my-first-app",
  "version": "0.0.1",
  "scripts": {
    "build": "devvit build",
    "dev": "devvit dev"
  },
  "dependencies": {
    "@devvit/public-api": "^0.10.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0"
  }
}
```

The key dependency is `@devvit/public-api`, which provides all Devvit functionality.

### `tsconfig.json` - TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "node",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "jsx": "react"
  },
  "include": ["src/**/*"]
}
```

Configured for modern TypeScript with strict type checking.

### `src/main.tsx` - Entry Point

```typescript
import { Devvit } from '@devvit/public-api';

Devvit.configure({
  redditAPI: true,
});

// Your app code goes here

export default Devvit;
```

This is where you'll write your app logic.

## Step 8: Verify Your Setup

Let's ensure everything works by running the development server.

### Start the dev server:

```bash
npm run dev
```

or

```bash
npx devvit dev
```

You should see output like:

```
✓ Building app...
✓ App built successfully
✓ Starting development server...
✓ Server running at http://localhost:3000

Watching for changes...
```

**What this does:**
- Compiles your TypeScript code
- Watches for file changes
- Runs a local preview environment
- Hot-reloads when you save files

### Access the development playground:

Open your browser to:
```
http://localhost:3000
```

You'll see the **Devvit Playground** - a local environment for testing your app without installing it on a real subreddit.

### Stop the dev server:

Press `Ctrl+C` in the terminal.

## Step 9: Understanding CLI Commands

Here are the essential Devvit CLI commands you'll use:

### Development Commands:

```bash
# Initialize and create a new app (via web wizard)
# Visit https://developers.reddit.com/new and follow the wizard
# The wizard will provide a command like:
npx devvit init <your-code>

# Start development server (from within your project directory)
npx devvit dev
# or use the npm script:
npm run dev

# Build your app
npx devvit build

# Run type checking
npx devvit check
```

**Note:** All `devvit` commands can be run with `npx devvit` (no global install needed). If you installed devvit globally, you can omit the `npx` prefix.

### Deployment Commands:

```bash
# Upload your app to Reddit
npx devvit upload

# Install app on a subreddit
npx devvit install <subreddit>

# List your apps
npx devvit list apps

# View app logs
npx devvit logs <app-name>
```

### Account Commands:

```bash
# Login to Reddit (usually not needed - wizard handles this)
npx devvit login

# Check who you're logged in as
npx devvit whoami

# Logout
npx devvit logout
```

### Help Commands:

```bash
# General help
npx devvit help

# Command-specific help
npx devvit help <command>
```

## Step 10: Configure Git (Optional but Recommended)

Version control is essential for managing your code.

### Initialize Git:

```bash
git init
git add .
git commit -m "Initial commit"
```

The template includes a `.gitignore` that excludes:
- `node_modules/`
- Build artifacts
- Local configuration files

### Create a GitHub repository (optional):

```bash
# Create repo on GitHub, then:
git remote add origin https://github.com/yourusername/my-first-app.git
git push -u origin main
```

## Common Setup Issues

### Issue: `Invalid code provided` error

**Solution:**
- This means the authentication code has expired or is invalid
- Visit `https://developers.reddit.com/new` to get a fresh code
- The wizard provides a one-time code that expires quickly
- Copy and paste the exact command the wizard provides

### Issue: `command not found: npx`

**Solution:**
- npx comes with npm, so update npm: `npm install -g npm@latest`
- Verify Node.js version: `node --version` (should be 18+)
- Restart your terminal after updating npm

### Issue: TypeScript errors in VS Code

**Solution:**
- Run `npm install` to install dependencies
- Reload VS Code: `Cmd/Ctrl + Shift + P` → "Reload Window"
- Ensure TypeScript version matches project

### Issue: `npx devvit dev` fails to start

**Solution:**
- Check that port 3000 isn't already in use
- Kill process using port: `lsof -ti:3000 | xargs kill` (Mac/Linux)
- Or specify different port: `npx devvit dev --port 3001`
- Alternatively, use the npm script: `npm run dev`

### Issue: Login redirect doesn't work

**Solution:**
- Ensure you're logged into Reddit in your browser
- Check that pop-ups aren't blocked
- Try manually copying the auth URL

## Environment Checklist

Verify your setup is complete:

- [ ] Node.js 18+ installed (`node --version`)
- [ ] npm installed and working (`npm --version`)
- [ ] npx available (`npx --version`)
- [ ] VS Code installed with recommended extensions
- [ ] Visited `https://developers.reddit.com/new` and completed wizard
- [ ] Project created and authenticated successfully
- [ ] Dependencies installed (wizard does this automatically)
- [ ] Can navigate to project directory (`cd my-first-app`)
- [ ] Dev server runs successfully (`npm run dev` or `npx devvit dev`)
- [ ] Can access playground at `http://localhost:3000`
- [ ] Git initialized (optional)

**Note:** You do NOT need to install Devvit CLI globally - npx handles everything!

## Best Practices for Development Environment

1. **Keep Node.js updated** - Use LTS versions for stability
2. **Use a version manager** - nvm makes switching Node versions easy
3. **Use npx** - Avoids global package installations and always uses latest versions
4. **Editor configuration** - Consistent formatting prevents merge conflicts
5. **Git from the start** - Initialize git early, commit often
6. **Separate workspace** - Create a dedicated folder for Devvit projects

## Next Steps

With your environment set up, you're ready to build your first Devvit app!

## Key Takeaways

1. **Node.js 18+** is required for Devvit development
2. **No global installation needed** - npx handles running Devvit on-demand
3. **Web wizard** at `developers.reddit.com/new` is the starting point for creating projects
4. **Wizard provides authentication** and generates project creation commands
5. **Authentication codes** are one-time use and expire quickly
6. **Project structure** follows standard TypeScript patterns
7. **Dev server** provides local testing environment
8. **VS Code** offers the best TypeScript development experience

## Checkpoint Questions

1. What is the minimum Node.js version required for Devvit?
2. What is the first step to create a new Devvit project?
3. Where does your app's code live in the project structure?
4. What command starts the local development server?
5. What URL is used to access the local playground?

<details>
<summary>Click to see answers</summary>

1. Node.js 18.x or higher
2. Visit `https://developers.reddit.com/new` and complete the web wizard
3. `src/main.tsx` (or `src/main.ts`)
4. `npx devvit dev` or `npm run dev`
5. `http://localhost:3000`

</details>

## Additional Resources

- [Devvit CLI Documentation](https://developers.reddit.com/docs/cli)
- [Node.js Installation Guide](https://nodejs.org/en/download/)
- [VS Code TypeScript Tutorial](https://code.visualstudio.com/docs/typescript/typescript-tutorial)
- [Git Basics](https://git-scm.com/book/en/v2/Getting-Started-Git-Basics)

## Practice Exercise

Try these to solidify your setup:

1. Create three different projects using different templates (visit the wizard each time)
2. Start and stop the dev server multiple times (`npm run dev`)
3. Make a small change to `main.tsx` and observe hot-reload
4. Run `npx devvit check` to see the type checker in action
5. Explore the playground interface at `localhost:3000`

---

➡️ **Continue to [Lesson 3: Your First Devvit App](./03-first-app.md)**

**Estimated time to complete**: 45 minutes
**Prerequisites**: Node.js basics, command line familiarity
