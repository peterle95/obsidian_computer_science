---
memory: to_finish
tags:
  - to_learn
language:
  - Core Concepts
review-date: 2025-11-20
last-reviewed: 2025-10-18
scheda: done
visit-count: 3
confidence-level: 1.5
consecutive-correct: 1
last-struggle-date: 2025-10-06
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
Package.json solves the fundamental problem of dependency management and project metadata organization in Node.js projects. Without it, developers would have to ==manually track and manage all external libraries, their versions, installation commands, build scripts, and project information==. This leads to =="dependency hell" where different developers have different versions of libraries, making projects unreproducible and difficult to maintain.==

Package.json is crucial because it enables reproducible builds, automated dependency installation, script automation, and serves as the single source of truth for project configuration. It's the foundation of the npm ecosystem and makes Node.js projects portable, shareable, and maintainable. Without package.json, collaboration on JavaScript projects would be nearly impossible at scale.

# **Core Explanation:**
---
Package.json is a JSON file that serves as the ==manifest for Node.js projects==. It contains metadata about the project and defines dependencies, scripts, and configuration settings. Every Node.js project should have a package.json file in its root directory.

**Key Components:**

**Basic Metadata:**

- `name`: Package name (required for publishable packages)
- `version`: Current version following semantic versioning
- `description`: Brief description of the project
- `author`: Project author information
- `license`: License type (MIT, GPL, etc.)
- `keywords`: Array of keywords for discoverability

**Dependencies:**

- `dependencies`: ==Runtime dependencies required for production==
- `devDependencies`: ==Development-only dependencies (testing, build tools)==
- `peerDependencies`: Dependencies that should be provided by the consuming project
- `optionalDependencies`: Optional dependencies that ==won't fail installation if unavailable==

**Scripts:**

- `scripts`: ==Custom commands that can be run== with `npm run <script-name>`
- Common scripts: `start`, `test`, `build`, `dev`, `lint`

**Configuration:**

- `main`: Entry point file for the package
- `engines`: Specifies Node.js and npm version requirements
- `repository`: Git repository information
- `bugs`: URL for bug reports
- `homepage`: Project homepage URL

**Advanced Features:**

- `bin`: Executable scripts to install globally
- `files`: Array of files to include when publishing
- `workspaces`: Monorepo workspace configuration
- `type`: Module type ("module" for ES modules, "commonjs" for CommonJS)

**Version Management:** Package.json uses semantic versioning (semver) with version ranges:

- `^1.2.3`: Compatible with version 1.2.3, allows minor and patch updates
- `~1.2.3`: Compatible with version 1.2.3, allows only patch updates
- `>=1.2.3`: Any version greater than or equal to 1.2.3
- `1.2.3`: Exact version match

# **Related Concepts:**

---
**NPM (Node Package Manager)** - The package manager that reads and processes package.json files. NPM uses package.json to install dependencies, run scripts, and manage project lifecycle.

**Package-lock.json** - Auto-generated file that locks exact dependency versions for reproducible installations. Works alongside package.json to ensure consistent dependency trees across environments.

**Semantic Versioning (SemVer)** - Version numbering scheme (MAJOR.MINOR.PATCH) used in package.json to manage dependency compatibility and breaking changes.

**Node_modules** - Directory where NPM installs dependencies listed in package.json. The structure is determined by package.json dependency specifications.

**Yarn.lock / PNPM-lock.yaml** - Alternative package managers' lock files that serve similar purposes to package-lock.json but with different algorithms and features.

**Manifest Files in Other Ecosystems** - Similar concept exists in other languages: requirements.txt (Python), Gemfile (Ruby), pom.xml (Java Maven), Cargo.toml (Rust), composer.json (PHP).

**Monorepo Workspaces** - Package.json supports workspace configuration for managing multiple related packages in a single repository.

**ES Modules vs CommonJS** - Package.json's `type` field determines how Node.js interprets JavaScript files (import/export vs require/module.exports).

**CI/CD Integration** - Package.json scripts are commonly used in continuous integration pipelines for automated testing, building, and deployment.

# **Examples:**
---

```json
// Basic package.json for a Node.js web application
{
  // Required fields for publishable packages
  "name": "my-web-app",
  "version": "1.0.0",
  "description": "A sample web application built with Express.js",
  
  // Project metadata
  "author": "John Doe <john@example.com>",
  "license": "MIT",
  "keywords": ["web", "express", "api", "nodejs"],
  
  // Entry point - file that runs when package is required
  "main": "src/app.js",
  
  // Custom scripts that can be run with npm run <script-name>
  "scripts": {
    "start": "node src/app.js",           // Production start command
    "dev": "nodemon src/app.js",          // Development with auto-restart
    "test": "jest",                       // Run tests
    "test:watch": "jest --watch",         // Run tests in watch mode
    "lint": "eslint src/**/*.js",         // Code linting
    "lint:fix": "eslint src/**/*.js --fix", // Auto-fix linting issues
    "build": "npm run lint && npm test",  // Pre-deployment checks
    "docker:build": "docker build -t my-web-app .", // Docker build
    "postinstall": "echo 'Dependencies installed successfully'" // Runs after npm install
  },
  
  // Production dependencies - required at runtime
  "dependencies": {
    "express": "^4.18.2",        // Web framework, allows minor updates
    "mongoose": "~7.0.3",        // MongoDB ODM, only patch updates
    "dotenv": "16.0.3",          // Environment variables, exact version
    "cors": "^2.8.5",            // Cross-origin resource sharing
    "helmet": "^6.1.5"           // Security middleware
  },
  
  // Development dependencies - only needed during development
  "devDependencies": {
    "nodemon": "^2.0.22",        // Auto-restart server during development
    "jest": "^29.5.0",           // Testing framework
    "supertest": "^6.3.3",       // HTTP assertion library for testing
    "eslint": "^8.39.0",         // Code linting
    "eslint-config-airbnb-base": "^15.0.0", // Airbnb ESLint rules
    "@types/node": "^18.16.3"    // TypeScript type definitions
  },
  
  // Node.js and npm version requirements
  "engines": {
    "node": ">=16.0.0",          // Minimum Node.js version
    "npm": ">=8.0.0"             // Minimum npm version
  },
  
  // Repository information
  "repository": {
    "type": "git",
    "url": "https://github.com/johndoe/my-web-app.git"
  },
  
  // Bug tracking
  "bugs": {
    "url": "https://github.com/johndoe/my-web-app/issues"
  },
  
  // Project homepage
  "homepage": "https://github.com/johndoe/my-web-app#readme"
}
```

```json
// Package.json for a publishable npm library
{
  "name": "@mycompany/utility-library",  // Scoped package name
  "version": "2.1.0",                    // Semantic version
  "description": "A collection of utility functions for JavaScript projects",
  
  // Library-specific fields
  "main": "dist/index.js",               // CommonJS entry point
  "module": "dist/index.esm.js",         // ES modules entry point
  "types": "dist/index.d.ts",            // TypeScript definitions
  "exports": {                           // Modern export map
    ".": {
      "import": "./dist/index.esm.js",   // ES module import
      "require": "./dist/index.js",      // CommonJS require
      "types": "./dist/index.d.ts"       // TypeScript types
    }
  },
  
  // Files to include when publishing to npm
  "files": [
    "dist",           // Built files
    "README.md",      // Documentation
    "LICENSE"         // License file
  ],
  
  // Scripts for library development and publishing
  "scripts": {
    "build": "rollup -c",                     // Build with Rollup
    "build:watch": "rollup -c --watch",       // Build in watch mode
    "test": "jest --coverage",                // Run tests with coverage
    "test:ci": "jest --coverage --ci",        // CI-optimized testing
    "lint": "eslint src --ext .ts,.js",       // Lint source code
    "typecheck": "tsc --noEmit",              // TypeScript type checking
    "prepublishOnly": "npm run build && npm test", // Pre-publish hooks
    "docs": "typedoc src/index.ts"            // Generate documentation
  },
  
  // No runtime dependencies for utility library
  "dependencies": {},
  
  // Development tools
  "devDependencies": {
    "@rollup/plugin-typescript": "^11.1.0",  // TypeScript bundling
    "@types/jest": "^29.5.0",                // Jest type definitions
    "jest": "^29.5.0",                       // Testing framework
    "rollup": "^3.21.0",                     // Module bundler
    "typescript": "^5.0.4",                  // TypeScript compiler
    "typedoc": "^0.24.6"                     // Documentation generator
  },
  
  // Peer dependencies - should be provided by consuming project
  "peerDependencies": {
    "lodash": "^4.17.0"          // Optional peer dependency
  },
  
  // Browser compatibility
  "browserslist": [
    "> 1%",                      // Browsers with >1% market share
    "last 2 versions",           // Last 2 versions of each browser
    "not dead"                   // Not dead browsers
  ],
  
  // Publishing configuration
  "publishConfig": {
    "access": "public",          // Public scoped package
    "registry": "https://registry.npmjs.org/"
  }
}
```

```json
// Package.json for a monorepo with workspaces
{
  "name": "my-monorepo",
  "version": "1.0.0",
  "description": "Monorepo containing multiple related packages",
  "private": true,                        // Don't publish root package
  
  // Workspace configuration
  "workspaces": [
    "packages/*",                         // All packages in packages/ directory
    "apps/*",                             // All apps in apps/ directory
    "tools/build-utils"                   // Specific tool package
  ],
  
  // Root-level scripts for entire monorepo
  "scripts": {
    "bootstrap": "npm install",           // Install all dependencies
    "build": "npm run build --workspaces", // Build all workspaces
    "test": "npm run test --workspaces",   // Test all workspaces
    "lint": "npm run lint --workspaces",   // Lint all workspaces
    "clean": "rm -rf node_modules packages/*/node_modules apps/*/node_modules",
    
    // Workspace-specific commands
    "build:api": "npm run build --workspace=@mycompany/api",
    "test:frontend": "npm run test --workspace=@mycompany/frontend",
    "dev:all": "concurrently \"npm run dev --workspace=@mycompany/api\" \"npm run dev --workspace=@mycompany/frontend\""
  },
  
  // Shared development dependencies for all workspaces
  "devDependencies": {
    "concurrently": "^8.0.1",            // Run multiple commands
    "lerna": "^6.6.1",                   // Monorepo management tool
    "eslint": "^8.39.0",                 // Shared linting
    "prettier": "^2.8.8",                // Code formatting
    "jest": "^29.5.0",                   // Shared testing framework
    "typescript": "^5.0.4"               // Shared TypeScript
  },
  
  // Common configuration for all workspaces
  "eslintConfig": {
    "extends": ["eslint:recommended"],
    "env": {
      "node": true,
      "es2022": true
    }
  },
  
  // Engine requirements for entire monorepo
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=9.0.0"
  }
}
```

```bash
# Command examples showing how package.json is used

# Initialize new package.json interactively
npm init

# Initialize with defaults
npm init -y

# Install dependencies (adds to package.json)
npm install express          # Adds to dependencies
npm install --save-dev jest  # Adds to devDependencies
npm install --save-peer react # Adds to peerDependencies

# Install exact versions
npm install lodash@4.17.21 --save-exact

# Install dependencies from package.json
npm install                  # Installs all dependencies

# Run custom scripts defined in package.json
npm run dev                  # Runs the "dev" script
npm run build               # Runs the "build" script
npm test                    # Runs the "test" script (npm shorthand)
npm start                   # Runs the "start" script (npm shorthand)

# Update package versions
npm update                  # Updates all packages within semver ranges
npm audit                   # Check for security vulnerabilities
npm audit fix              # Automatically fix security issues

# Workspace commands (for monorepos)
npm run build --workspaces  # Run build in all workspaces
npm install --workspace=@mycompany/api # Install deps for specific workspace
```

# **Flashcards:**
---

What is package.json and what problem does it solve?;; Package.json is a JSON manifest file for Node.js projects that solves dependency management and project metadata organization, enabling reproducible builds and automated dependency installation.

What's the difference between dependencies and devDependencies in package.json?;; Dependencies are runtime libraries required for production, while devDependencies are development-only tools (testing, build tools, linters) not needed in production environments.

What do the version prefixes ^, ~, and exact versions mean in package.json?;; ^ allows compatible minor and patch updates (^1.2.3 allows 1.x.x), ~ allows only patch updates (~1.2.3 allows 1.2.x), and exact versions (1.2.3) install that specific version only.

What is the purpose of the scripts section in package.json?;; The scripts section defines custom commands that can be executed with npm run <script-name>, commonly used for build processes, testing, linting, and development tasks.

How do package.json and package-lock.json work together?;; Package.json defines dependency requirements and version ranges, while package-lock.json locks exact versions and dependency trees to ensure reproducible installations across different environments.

What are workspaces in package.json and when are they used?;; Workspaces enable monorepo management by allowing multiple related packages to be managed from a single root package.json, sharing dependencies and enabling coordinated development of related projects.