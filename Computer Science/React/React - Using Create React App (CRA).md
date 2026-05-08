---
memory: to_finish
tags:
  - will_learn
language:
  - React
review-date:
last-reviewed: ""
scheda: done
visit-count: 0
confidence-level: 1
consecutive-correct: 0
last-struggle-date: ""
cssclasses:
---

```dataviewjs
const currentPage = dv.current();
let visitCount = currentPage.file.frontmatter["visit-count"] || 0;
let confidence = currentPage.file.frontmatter["confidence-level"] || 1;
let streak = currentPage.file.frontmatter["consecutive-correct"] || 0;

const container = this.container.createEl('div');
container.style.cssText = `
    background: #2a2a2a; border: 1px solid #404040; border-radius: 6px;
    padding: 12px; margin: 10px 0; display: inline-block;
`;

// Status display
const status = container.createEl('div');
status.innerHTML = `
    <strong>Learning Progress</strong><br>
    Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
`;
status.style.cssText = 'margin-bottom: 10px; font-size: 13px; color: #cccccc;';

// Quick feedback buttons
const buttonContainer = container.createEl('div');
['Got it! ✅', 'Struggled ⚠️', 'Failed ❌'].forEach((label, index) => {
    const btn = buttonContainer.createEl('button');
    btn.textContent = label;
    btn.style.cssText = `
        margin-right: 8px; padding: 4px 8px; border: none; border-radius: 3px;
        cursor: pointer; font-size: 11px;
        background: ${['#28a745', '#ffc107', '#dc3545'][index]};
        color: ${index === 1 ? '#000' : '#fff'};
    `;
    
    btn.addEventListener('click', async () => {
        visitCount++;
        if (index === 0) { // Got it
            confidence = Math.min(5, confidence + 0.5);
            streak++;
        } else { // Struggled or failed
            confidence = Math.max(1, confidence - 0.5);
            streak = 0;
        }
        
        const file = app.vault.getAbstractFileByPath(currentPage.file.path);
        await app.fileManager.processFrontMatter(file, (fm) => {
            fm["visit-count"] = visitCount;
            fm["confidence-level"] = confidence;
            fm["consecutive-correct"] = streak;
            fm["last-reviewed"] = new Date().toISOString().split('T')[0];
            if (index > 0) fm["last-struggle-date"] = new Date().toISOString().split('T')[0];
        });
        
        status.innerHTML = `
            <strong>Learning Progress</strong><br>
            Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
        `;
    });
});
```

```dataviewjs
// Get all flashcards from the current note
const currentPage = dv.current();
const content = await app.vault.read(app.vault.getAbstractFileByPath(currentPage.file.path));

// Split content into lines
const lines = content.split('\n');
let flashcardLines = [];
let inCodeBlock = false;

// Collect all potential flashcard lines - simplified approach
for (let i = 0; i < lines.length; i++) {
    const line = lines[i];
    
    // Track code blocks
    if (line.trim().startsWith('```')) {
        inCodeBlock = !inCodeBlock;
        continue;
    }
    
    // Skip everything inside code blocks
    if (inCodeBlock) continue;
    
    // Check if line contains flashcard separator
    if (line.includes(';;')) {
        flashcardLines.push(line);
    }
}

const filteredLines = flashcardLines.filter(line => {
    // Only filter out lines that are clearly part of the JavaScript code
    // Be more specific with patterns to avoid false positives
    return !(
        line.trim().startsWith('const ') ||
        line.trim().startsWith('let ') ||
        line.trim().startsWith('function ') ||
        line.trim().startsWith('return ') ||
        line.trim().startsWith('if (') ||
        line.trim().startsWith('for (') ||
        line.trim().startsWith('while (') ||
        line.includes('dataviewjs') ||
        line.includes('content.split') ||
        line.includes('flashcardLines') ||
        line.includes('this.container') ||
        line.includes('addEventListener') ||
        line.includes('console.log') ||
        /\.\w+\(/.test(line) && (line.includes('.map(') || line.includes('.filter(') || line.includes('.forEach(') || line.includes('.find('))
    );
});

// Process the flashcards
const flashcards = [];
for (let i = 0; i < filteredLines.length; i++) {
    const line = filteredLines[i];
    try {
        const separatorIndex = line.indexOf(';;');
        if (separatorIndex === -1) continue;
        
        const front = line.substring(0, separatorIndex).trim();
        const back = line.substring(separatorIndex + 2).trim();
        
        // Very minimal validation - just check they exist
        if (front && back) {
            flashcards.push({ 
                front: front, 
                back: back,
                index: i // Keep track of original order
            });
        }
    } catch (error) {
        console.log('Error processing flashcard line:', line, error);
    }
}

// Debug information
console.log('All lines with ;;:', flashcardLines.length);
console.log('After filtering:', filteredLines.length);
console.log('Filtered lines:', filteredLines);
console.log('Final flashcards:', flashcards.length);

if (flashcards.length === 0) {
    const errorMsg = this.container.createEl('div');
    errorMsg.style.cssText = 'background: #2a2a2a; padding: 15px; border-radius: 6px; color: #cccccc;';
    errorMsg.innerHTML = `
        <strong>No flashcards found!</strong><br><br>
        Lines with ';;' found: ${flashcardLines.length}<br>
        After filtering: ${filteredLines.length}<br>
        Valid flashcards: ${flashcards.length}<br><br>
        <strong>All lines with ;;:</strong><br>
        ${flashcardLines.map((line, i) => `${i+1}. ${line.substring(0, 50)}...`).join('<br>')}
    `;
    return;
}

// Flashcard state
let currentCardIndex = 0;
let showingBack = false;
let correctCount = 0;
let totalReviewed = 0;

// Create main container
const container = this.container.createEl('div');
container.style.cssText = `
    background: #2a2a2a;
    border: 1px solid #404040;
    border-radius: 8px;
    padding: 20px;
    margin: 15px 0;
    max-width: 700px;
`;

// Header with progress
const header = container.createEl('div');
header.style.cssText = 'display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px;';

const title = header.createEl('h3');
title.textContent = `Flashcards (${flashcards.length} total)`;
title.style.cssText = 'margin: 0; color: #ffffff;';

const progress = header.createEl('div');
progress.style.cssText = 'color: #cccccc; font-size: 14px; text-align: right;';

// Card container
const cardContainer = container.createEl('div');
cardContainer.style.cssText = `
    background: #1a1a1a;
    border: 2px solid #404040;
    border-radius: 6px;
    padding: 30px;
    margin: 20px 0;
    min-height: 120px;
    display: flex;
    align-items: center;
    justify-content: center;
    text-align: center;
    cursor: pointer;
    transition: all 0.2s;
`;

const cardText = cardContainer.createEl('div');
cardText.style.cssText = `
    font-size: 16px;
    line-height: 1.5;
    color: #ffffff;
    word-wrap: break-word;
    max-width: 100%;
`;

// Button container
const buttonContainer = container.createEl('div');
buttonContainer.style.cssText = 'display: flex; gap: 10px; justify-content: center; flex-wrap: wrap; margin-bottom: 15px;';

// Control buttons
const flipButton = buttonContainer.createEl('button');
flipButton.textContent = 'Flip Card';
flipButton.style.cssText = `
    background: #4a9eff; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px; font-weight: 500;
`;

const easyButton = buttonContainer.createEl('button');
easyButton.textContent = 'Easy ✅';
easyButton.style.cssText = `
    background: #28a745; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const hardButton = buttonContainer.createEl('button');
hardButton.textContent = 'Hard ❌';
hardButton.style.cssText = `
    background: #dc3545; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const nextButton = buttonContainer.createEl('button');
nextButton.textContent = 'Next →';
nextButton.style.cssText = `
    background: #6c757d; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const prevButton = buttonContainer.createEl('button');
prevButton.textContent = '← Prev';
prevButton.style.cssText = `
    background: #6c757d; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const shuffleButton = buttonContainer.createEl('button');
shuffleButton.textContent = '🔀 Shuffle';
shuffleButton.style.cssText = `
    background: #17a2b8; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

// Functions
function updateDisplay() {
    const card = flashcards[currentCardIndex];
    cardText.textContent = showingBack ? card.back : card.front;
    
    progress.innerHTML = `Card ${currentCardIndex + 1} of ${flashcards.length}`;
    if (totalReviewed > 0) {
        progress.innerHTML += `<br>Correct: ${correctCount}/${totalReviewed} (${Math.round(correctCount/totalReviewed*100)}%)`;
    }
    
    // Show/hide difficulty buttons
    if (showingBack) {
        easyButton.style.display = 'inline-block';
        hardButton.style.display = 'inline-block';
        flipButton.textContent = 'Show Front';
        cardContainer.style.borderColor = '#ffc107';
        cardContainer.style.backgroundColor = '#252525';
    } else {
        easyButton.style.display = 'none';
        hardButton.style.display = 'none';
        flipButton.textContent = 'Flip Card';
        cardContainer.style.borderColor = '#404040';
        cardContainer.style.backgroundColor = '#1a1a1a';
    }
    
    // Update navigation buttons
    prevButton.style.display = currentCardIndex > 0 ? 'inline-block' : 'none';
    nextButton.textContent = currentCardIndex < flashcards.length - 1 ? 'Next →' : 'Restart';
}

function flipCard() {
    showingBack = !showingBack;
    updateDisplay();
}

function nextCard() {
    if (currentCardIndex < flashcards.length - 1) {
        currentCardIndex++;
    } else {
        currentCardIndex = 0;
    }
    showingBack = false;
    updateDisplay();
}

function prevCard() {
    if (currentCardIndex > 0) {
        currentCardIndex--;
        showingBack = false;
        updateDisplay();
    }
}

function markCorrect() {
    if (showingBack) {
        correctCount++;
        totalReviewed++;
        nextCard();
    }
}

function markIncorrect() {
    if (showingBack) {
        totalReviewed++;
        nextCard();
    }
}

function shuffleCards() {
    for (let i = flashcards.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [flashcards[i], flashcards[j]] = [flashcards[j], flashcards[i]];
    }
    currentCardIndex = 0;
    showingBack = false;
    correctCount = 0;
    totalReviewed = 0;
    updateDisplay();
}

// Event listeners
cardContainer.addEventListener('click', flipCard);
flipButton.addEventListener('click', flipCard);
easyButton.addEventListener('click', markCorrect);
hardButton.addEventListener('click', markIncorrect);
nextButton.addEventListener('click', nextCard);
prevButton.addEventListener('click', prevCard);
shuffleButton.addEventListener('click', shuffleCards);

// Instructions
const instructions = container.createEl('div');
instructions.style.cssText = 'font-size: 12px; color: #888; text-align: center; line-height: 1.4;';
instructions.innerHTML = `
    <strong>Controls:</strong> Click card to flip | Navigation buttons | Easy/Hard to mark
`;

// Initialize
updateDisplay();
```

# **Purpose/Why**:
---

Create React App (CRA) solves the problem of complex build configuration and tooling setup required for modern React applications. Without CRA, developers need to manually configure Webpack, Babel, ESLint, testing frameworks, and numerous other tools—a time-consuming and error-prone process that requires deep knowledge of the JavaScript ecosystem.

CRA provides a pre-configured, zero-configuration development environment that includes everything needed to build a production-ready React application: a development server with hot reloading, a production build optimizer, JSX and modern JavaScript transpilation, CSS preprocessing support, and a testing setup. This allows developers to focus on writing application code rather than maintaining build configurations.

It's important because it standardizes React project setup, reduces the learning curve for beginners, ensures best practices are followed by default, and provides a consistent development experience across teams. CRA democratized React development by making it accessible to developers without build tool expertise.

# **Core Explanation:**
---

Create React App is an officially supported command-line tool that scaffolds a new React project with a pre-configured development environment. It's maintained by the React team and the community.

**Key Characteristics:**

- **Zero Configuration**: Works out of the box without needing to configure Webpack, Babel, or other tools
- **Single Dependency**: All build tools are abstracted into a single dependency (`react-scripts`)
- **Opinionated Setup**: Enforces best practices and sensible defaults
- **Ejectable**: Allows you to "eject" to expose and customize all configuration files if needed (one-way operation)
- **Built-in Features**: Includes development server, hot module replacement, production build optimization, CSS and file importing, environment variables, testing with Jest, and more

**How It Works:**

1. When you run `npx create-react-app my-app`, it creates a new directory with a basic React project structure
2. The project includes `react-scripts` as a dependency, which contains all the build configuration and scripts
3. Running `npm start` launches a development server with hot reloading
4. Running `npm run build` creates an optimized production build in the `build` folder
5. The configuration (Webpack, Babel, etc.) is hidden inside `react-scripts`, keeping your project clean
6. Updates to build tools are handled by updating the `react-scripts` version

**Project Structure:**

```
my-app/
├── node_modules/
├── public/
│   ├── index.html
│   └── favicon.ico
├── src/
│   ├── App.js
│   ├── App.css
│   ├── index.js
│   └── index.css
├── package.json
└── README.md
```

# **Related Concepts:**
---


**[[React - Vite]]**: A modern, faster alternative to CRA that uses native ES modules and provides instant server start. Unlike CRA's Webpack-based approach, Vite uses esbuild for faster builds and is increasingly preferred for new projects.

**[[Next.js - Full-Stack React Framework]]**: A React framework that provides more features than CRA, including server-side rendering, static site generation, file-based routing, and API routes. It's more opinionated and feature-rich than CRA.

**Webpack**: The underlying bundler that CRA uses internally. Understanding Webpack helps when you need to eject from CRA or migrate to custom configurations.

**[[JavaScript Transpilers (e.g., Babel) and Polyfills - Overview]]**: The JavaScript transpiler CRA uses to convert modern JavaScript and JSX into browser-compatible code. CRA configures Babel automatically.

**react-scripts**: The package that contains all of CRA's configuration and build scripts. It's the single dependency that powers CRA's magic.

**[[npm]]/npx**: Package management tools. `npx` allows running CRA without installing it globally, ensuring you always use the latest version.

**[[React - What is a Single Page Application (SPA)]]**: CRA is designed specifically for building SPAs, where the application runs entirely in the browser with client-side routing.

# **Examples:**
---

```bash
# EXAMPLE 1: Creating a new React app with CRA
# Using npx ensures you're using the latest version without global installation
npx create-react-app my-app

# Navigate into the newly created project directory
cd my-app

# Start the development server on http://localhost:3000
# This command launches a dev server with hot reloading
# Any changes to files in src/ will automatically refresh the browser
npm start

# Create an optimized production build
# This minifies code, optimizes assets, and creates a build/ folder
# The output is ready to be deployed to a web server
npm run build

# Run the test suite in interactive watch mode
# CRA includes Jest and React Testing Library by default
npm test
```

```javascript
// EXAMPLE 2: Basic CRA project structure and entry point
// File: src/index.js
// This is the entry point of every CRA application
// It renders the root React component into the DOM

import React from 'react';
import ReactDOM from 'react-dom/client';
import './index.css'; // Global CSS can be imported directly
import App from './App'; // Import the main App component

// Create a root for React 18's concurrent features
// The element with id 'root' is defined in public/index.html
const root = ReactDOM.createRoot(document.getElementById('root'));

// Render the App component
// StrictMode helps identify potential problems in the application
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

```javascript
// EXAMPLE 3: Main App component in CRA
// File: src/App.js
// This is the root component of your application
// CRA allows importing CSS files directly into JavaScript

import React, { useState } from 'react';
import './App.css'; // Component-specific CSS
import logo from './logo.svg'; // You can import images as file paths

function App() {
  // State management works normally in CRA projects
  const [count, setCount] = useState(0);

  return (
    <div className="App">
      <header className="App-header">
        {/* Images imported as modules are automatically processed by Webpack */}
        <img src={logo} className="App-logo" alt="logo" />
        
        <h1>Welcome to React with CRA</h1>
        
        {/* Example of interactive component */}
        <p>Button clicked {count} times</p>
        <button onClick={() => setCount(count + 1)}>
          Click me
        </button>
      </header>
    </div>
  );
}

export default App;
```

```javascript
// EXAMPLE 4: Using environment variables in CRA
// File: src/config.js
// CRA supports environment variables that must be prefixed with REACT_APP_
// Create a .env file in the root directory to define variables

// .env file content:
// REACT_APP_API_URL=https://api.example.com
// REACT_APP_API_KEY=your-api-key-here

// Accessing environment variables in code
// These are embedded at build time, not runtime
const config = {
  // Environment variables are accessed via process.env
  // They are available anywhere in your src/ code
  apiUrl: process.env.REACT_APP_API_URL || 'http://localhost:3000',
  apiKey: process.env.REACT_APP_API_KEY,
  
  // NODE_ENV is set automatically by CRA
  // It's 'development' for npm start, 'production' for npm run build
  isDevelopment: process.env.NODE_ENV === 'development'
};

export default config;
```

```javascript
// EXAMPLE 5: Importing different file types in CRA
// File: src/components/MediaComponent.js
// CRA's Webpack configuration allows importing various file types

import React from 'react';

// Import CSS modules for scoped styling (file must end in .module.css)
import styles from './MediaComponent.module.css';

// Import regular CSS (globally applied)
import './global.css';

// Import images - Webpack returns the final path after optimization
import heroImage from '../assets/hero.jpg';

// Import SVGs as React components (useful for icons)
import { ReactComponent as Logo } from '../assets/logo.svg';

// Import JSON files directly as JavaScript objects
import configData from '../config.json';

function MediaComponent() {
  return (
    <div className={styles.container}> {/* Scoped CSS class */}
      {/* Image imported as module - automatically optimized */}
      <img src={heroImage} alt="Hero" />
      
      {/* SVG imported as component - can be styled with props */}
      <Logo className={styles.logo} fill="blue" />
      
      {/* JSON data is directly accessible */}
      <p>App name: {configData.appName}</p>
    </div>
  );
}

export default MediaComponent;
```

```javascript
// EXAMPLE 6: Ejecting from CRA (advanced use case)
// Run this command with caution - it's irreversible!
// npm run eject

// This command copies all configuration files and dependencies into your project
// After ejecting, you'll have full control over:
// - Webpack configuration (config/webpack.config.js)
// - Babel configuration (.babelrc or babel.config.js)
// - ESLint configuration (.eslintrc)
// - All build scripts (scripts/ folder)

// File structure after ejecting:
/*
my-app/
├── config/
│   ├── webpack.config.js     // Full Webpack configuration
│   ├── jest.config.js         // Jest testing configuration
│   └── ...
├── scripts/
│   ├── build.js               // Production build script
│   ├── start.js               // Development server script
│   └── test.js                // Test runner script
├── src/
└── package.json               // Now contains all dependencies individually
*/

// Reasons to eject:
// - Need custom Webpack loaders or plugins
// - Require specific Babel plugins not supported by CRA
// - Need to modify build process significantly

// Alternatives to ejecting:
// - Use react-app-rewired or CRACO to override config without ejecting
// - Migrate to Vite or custom Webpack setup
// - Use Next.js if you need SSR or more features
```

# **Flashcards:**

---

What is Create React App (CRA) and what problem does it solve?;; Create React App is an officially supported CLI tool that scaffolds React projects with pre-configured build tools (Webpack, Babel, ESLint, Jest). It solves the problem of complex build configuration setup, allowing developers to start building React apps immediately without manually configuring tooling.

What are the main commands used in a CRA project and what does each do?;; `npm start` - launches development server with hot reloading; `npm run build` - creates optimized production build; `npm test` - runs test suite in watch mode; `npm run eject` - exposes all configuration files (irreversible).

What is the purpose of react-scripts in a CRA project?;; react-scripts is the single dependency that contains all of CRA's build configuration and scripts. It abstracts away Webpack, Babel, ESLint, and other tool configurations, keeping your project clean and manageable. Updates to build tools are handled by updating the react-scripts version.

How do environment variables work in Create React App?;; Environment variables must be prefixed with REACT_APP_ to be accessible (e.g., REACT_APP_API_URL). They are defined in .env files and accessed via process.env.REACT_APP_VARIABLE_NAME. They are embedded at build time, not runtime. NODE_ENV is automatically set to 'development' or 'production'.

What does "ejecting" from CRA mean and when should you consider it?;; Ejecting is a one-way operation (npm run eject) that exposes all hidden configuration files (Webpack, Babel, ESLint) into your project, giving you full control. Consider it only when you need custom configurations that CRA doesn't support. Alternatives include react-app-rewired, CRACO, or migrating to Vite/Next.js.

What file types can be imported in a CRA project and how are they handled?;; CRA allows importing: CSS files (global or modules with .module.css), images (returns optimized path), SVGs as React components (using ReactComponent import), and JSON files (as JavaScript objects). Webpack processes all these imports and optimizes them for production builds.