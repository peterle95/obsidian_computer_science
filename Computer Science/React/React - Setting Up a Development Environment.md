---
memory: to_finish
tags:
  - learned
language:
  - React
review-date:
last-reviewed:
scheda: done
---
# **Purpose/Why**:
---
A React development environment solves the problem of turning modern React source code into something the browser can run efficiently while giving the developer fast feedback during development.

React projects usually use features that browsers do not execute directly in the same form you write them: JSX, ES modules, package imports, TypeScript sometimes, CSS imports, and optimized production builds. A development environment connects all of this together.

The goal is to create a local workspace where you can:

- write components using JSX;
- install and manage dependencies;
- run a local development server;
- see changes quickly in the browser;
- catch syntax and build errors early;
- produce an optimized production build when the app is ready.

For learning React basics, a lightweight tool such as [[React - Vite]] is usually the clearest choice because it gives you a modern React setup without hiding too much behind framework conventions. For larger production apps, the official React docs generally recommend starting with a React framework, such as Next.js or React Router framework mode, because routing, data loading, rendering strategy, and deployment become important architectural choices.

# **Core Explanation:**
---
A React development environment is the collection of tools that lets you create, run, test, and build a React app locally.

The main pieces are:

1. **[[Node.js - Server-Side JavaScript]]**
   Node.js lets you run JavaScript tools on your computer outside the browser. React itself runs in the browser, but the tools around React, such as Vite, npm scripts, bundlers, and development servers, run through Node.js.

2. **[[npm]] or another package manager**
   A package manager installs libraries and tools. For example, it installs `react`, `react-dom`, `vite`, testing libraries, formatters, and other dependencies listed in `package.json`.

3. **Project scaffolding tool**
   A scaffolding tool creates the initial folder structure and configuration. With Vite, `npm create vite@latest` can generate a React project template.

4. **Build tool / bundler**
   A build tool processes your source files. It transforms JSX into JavaScript, resolves imports, bundles modules, handles CSS imports, and prepares optimized files for production. Vite is both a development server and a build tool.

5. **Development server**
   The development server runs the app locally, usually at a URL such as `http://localhost:5173/`. It watches your files and updates the browser quickly when you save changes.

6. **Source folder**
   In a typical Vite React project, most app code lives in `src/`. The important files are commonly:
   - `src/main.jsx` or `src/main.tsx`: the entry point where React attaches the app to the DOM;
   - `src/App.jsx` or `src/App.tsx`: the first main component;
   - `index.html`: the HTML file that contains the root element for the React app.

7. **npm scripts**
   `package.json` usually contains scripts such as:
   - `npm run dev`: start the local development server;
   - `npm run build`: create an optimized production build;
   - `npm run preview`: preview the production build locally.

A simple mental model:

You write React components in `src/` -> Vite reads the entry point -> React renders into the root DOM element -> the development server shows the app in the browser -> when you save files, Vite updates the running app quickly.

Important note: [[React - Using Create React App (CRA)]] used to be the standard beginner setup, but it is now deprecated. It is still useful to recognize older projects, but new learning projects should usually use [[React - Vite]] or a recommended React framework.


# **Memory Palace**
---
## **1. Encoded Imagery / Story (Visual OR Non-Visual)**
_Describe the mnemonic you attach to the spot. This can be visual, verbal, symbolic, conceptual, or sensory._
  

## **2. Retrieval Path**
_Write a clear retrieval route (e.g., “enter kitchen → sink → fridge → window”)._

# **Related Concepts:**
---
- **[[React - Vite]]**: A modern build tool and development server commonly used for learning React from scratch. It provides fast startup, fast updates, and a simple project structure.

- **[[React - Using Create React App (CRA)]]**: The older traditional setup for React projects. It is useful to understand because many existing tutorials and older projects use it, but it is deprecated and should usually not be chosen for new projects.

- **[[React - JSX (JavaScript XML)]]**: JSX is one of the main reasons React needs a build tool. The browser does not directly understand JSX, so the development environment transforms it into regular JavaScript.

- **[[React - What is React (The Why)]]**: React explains the UI model; the development environment explains the local toolchain that lets you write and run that model.

- **[[Next.js - Full-Stack React Framework]]**: A production React framework that adds routing, server rendering, data loading patterns, and deployment conventions on top of React.

# **Examples:**
---
## **Example 1: Create a new React project with Vite**
```bash
# Create a new project from Vite's templates.
# "my-react-app" is the folder that will be created.
# "--template react" tells Vite to use the React JavaScript template.
npm create vite@latest my-react-app -- --template react

# Move into the new project folder so commands run in the right place.
cd my-react-app

# Install the dependencies listed in package.json.
# This creates node_modules and prepares the project tools.
npm install

# Start the local development server.
# Vite will print a localhost URL, commonly http://localhost:5173/.
npm run dev
```

## **Example 2: Typical Vite React entry point**
```jsx
// Import React's StrictMode helper.
// StrictMode helps reveal potential problems during development.
import { StrictMode } from 'react';

// Import createRoot, the React DOM API that mounts a React app.
import { createRoot } from 'react-dom/client';

// Import the main App component.
// App is usually where the visible UI begins.
import App from './App.jsx';

// Find the real DOM element with id="root" in index.html.
// React will control the UI inside this element.
const rootElement = document.getElementById('root');

// Create a React root and render the App component into the page.
// From this point on, React manages the UI inside #root.
createRoot(rootElement).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

## **Example 3: Common package.json scripts**
```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  }
}
```

```bash
# Runs the app locally for development.
npm run dev

# Creates optimized production files, usually in the dist folder.
npm run build

# Serves the production build locally so you can inspect it before deploying.
npm run preview
```

# **Flashcards:**
---
What problem does a React development environment solve?;;It gives you the local tools needed to write JSX, manage dependencies, run a development server, get fast browser feedback, and build optimized production files.
What are the main tools in a basic React development environment?;;Node.js, a package manager such as npm, a scaffolding tool, a build tool such as Vite, a development server, source files, and package.json scripts.
Why is Vite commonly used for learning React today?;;Vite creates a modern React project with a simple structure, fast development server, quick updates, and less setup overhead than older tools like Create React App.