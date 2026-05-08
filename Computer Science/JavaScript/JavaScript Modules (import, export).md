---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date: ""
last-reviewed: 2025-08-19
scheda: done
visit-count: 4
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-07-20

---
```dataviewjs
const currentPage = dv.current;
let visitCount = currentPage.file.frontmatter["visit-count"] || 0;
let confidence = currentPage.file.frontmatter["confidence-level"] || 1;
let streak = currentPage.file.frontmatter["consecutive-correct"] || 0;

const container = this.container.createEl('div');
container.style.cssText = `
 background:

# 2a2a2a; border: 1px solid

# 404040; border-radius: 6px;
 padding: 12px; margin: 10px 0; display: inline-block;
`;

// Status display
const status = container.createEl('div');
status.innerHTML = `
 <strong>Learning Progress</strong><br>
 Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
`;
status.style.cssText = 'margin-bottom: 10px; font-size: 13px; color:

# cccccc;';

// Quick feedback buttons
const buttonContainer = container.createEl('div');
['Got it! ✅', 'Struggled ⚠️', 'Failed ❌'].forEach((label, index) => {
 const btn = buttonContainer.createEl('button');
 btn.textContent = label;
 btn.style.cssText = `
 margin-right: 8px; padding: 4px 8px; border: none; border-radius: 3px;
 cursor: pointer; font-size: 11px;
 background: ${['

# 28a745', '

# ffc107', '

# dc3545'][index]};
 color: ${index === 1 ? '

# 000' : '

# fff'};
 `;

 btn.addEventListener('click', async => {
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
 fm["last-reviewed"] = new Date.toISOString.split('T');
 if (index > 0) fm["last-struggle-date"] = new Date.toISOString.split('T');
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
const currentPage = dv.current;
const content = await app.vault.read(app.vault.getAbstractFileByPath(currentPage.file.path));

// Split content into lines
const lines = content.split('\n');
let flashcardLines = ;
let inCodeBlock = false;

// Collect all potential flashcard lines - simplified approach
for (let i = 0; i < lines.length; i++) {
 const line = lines[i];

 // Track code blocks
 if (line.trim.startsWith('```')) {
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
 line.trim.startsWith('const ') ||
 line.trim.startsWith('let ') ||
 line.trim.startsWith('function ') ||
 line.trim.startsWith('return ') ||
 line.trim.startsWith('if (') ||
 line.trim.startsWith('for (') ||
 line.trim.startsWith('while (') ||
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
const flashcards = ;
for (let i = 0; i < filteredLines.length; i++) {
 const line = filteredLines[i];
 try {
 const separatorIndex = line.indexOf(';;');
 if (separatorIndex === -1) continue;

 const front = line.substring(0, separatorIndex).trim;
 const back = line.substring(separatorIndex + 2).trim;

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
 errorMsg.style.cssText = 'background:

# 2a2a2a; padding: 15px; border-radius: 6px; color:

# cccccc;';
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
 background:

# 2a2a2a;
 border: 1px solid

# 404040;
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
title.style.cssText = 'margin: 0; color:

# ffffff;';

const progress = header.createEl('div');
progress.style.cssText = 'color:

# cccccc; font-size: 14px; text-align: right;';

// Card container
const cardContainer = container.createEl('div');
cardContainer.style.cssText = `
 background:

# 1a1a1a;
 border: 2px solid

# 404040;
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
 color:

# ffffff;
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
 background:

# 4a9eff; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; font-weight: 500;
`;

const easyButton = buttonContainer.createEl('button');
easyButton.textContent = 'Easy ✅';
easyButton.style.cssText = `
 background:

# 28a745; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const hardButton = buttonContainer.createEl('button');
hardButton.textContent = 'Hard ❌';
hardButton.style.cssText = `
 background:

# dc3545; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const nextButton = buttonContainer.createEl('button');
nextButton.textContent = 'Next →';
nextButton.style.cssText = `
 background:

# 6c757d; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const prevButton = buttonContainer.createEl('button');
prevButton.textContent = '← Prev';
prevButton.style.cssText = `
 background:

# 6c757d; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const shuffleButton = buttonContainer.createEl('button');
shuffleButton.textContent = '🔀 Shuffle';
shuffleButton.style.cssText = `
 background:

# 17a2b8; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

// Functions
function updateDisplay {
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
 cardContainer.style.borderColor = '

# ffc107';
 cardContainer.style.backgroundColor = '

# 252525';
 } else {
 easyButton.style.display = 'none';
 hardButton.style.display = 'none';
 flipButton.textContent = 'Flip Card';
 cardContainer.style.borderColor = '

# 404040';
 cardContainer.style.backgroundColor = '

# 1a1a1a';
 }

 // Update navigation buttons
 prevButton.style.display = currentCardIndex > 0 ? 'inline-block' : 'none';
 nextButton.textContent = currentCardIndex < flashcards.length - 1 ? 'Next →' : 'Restart';
}

function flipCard {
 showingBack = !showingBack;
 updateDisplay;
}

function nextCard {
 if (currentCardIndex < flashcards.length - 1) {
 currentCardIndex++;
 } else {
 currentCardIndex = 0;
 }
 showingBack = false;
 updateDisplay;
}

function prevCard {
 if (currentCardIndex > 0) {
 currentCardIndex--;
 showingBack = false;
 updateDisplay;
 }
}

function markCorrect {
 if (showingBack) {
 correctCount++;
 totalReviewed++;
 nextCard;
 }
}

function markIncorrect {
 if (showingBack) {
 totalReviewed++;
 nextCard;
 }
}

function shuffleCards {
 for (let i = flashcards.length - 1; i > 0; i--) {
 const j = Math.floor(Math.random * (i + 1));
 [flashcards[i], flashcards[j]] = [flashcards[j], flashcards[i]];
 }
 currentCardIndex = 0;
 showingBack = false;
 correctCount = 0;
 totalReviewed = 0;
 updateDisplay;
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
instructions.style.cssText = 'font-size: 12px; color:

# 888; text-align: center; line-height: 1.4;';
instructions.innerHTML = `
 <strong>Controls:</strong> Click card to flip | Navigation buttons | Easy/Hard to mark
`;

// Initialize
updateDisplay;
```

# **Purpose/Why**:

---
JavaScript modules solve the fundamental problem of ==code organization, reusability, and dependency management in JavaScript applications==. Before modules, JavaScript code was often written in a single file or included through script tags, leading to global namespace pollution, dependency management nightmares, and difficulty in maintaining large codebases. Modules provide a way to split code into separate files, explicitly define what each file exports and imports, create encapsulated scopes, and establish clear dependency relationships. This is crucial for building scalable, maintainable JavaScript applications, enabling code reuse across projects, and supporting modern development practices like component-based architecture and package management through npm.

# **Core Explanation:**

---
- **Imports and Exports**: How different modules (JavaScript files) share code with each other.
 - **Default Exports**: A single value or component exported as the primary export from a module (e.g., `export default HelpCenter;`).

 - **Named Exports**: Multiple values or components exported from a module, identified by their names (e.g., `export { HelpCenter };`).
 - **Import Syntax**: How to import default and named exports.
 >- `import HelpCenter from "./path/to/file";` (for default exports)
 >- `import { SomeNamedExport } from "./path/to/file";` (for named exports)
- **"Module not found: Error: Can't resolve" Error**: Occurs when an `import` statement tries to load a file that doesn't exist at the specified path. This often points to incorrect file paths or missing files.

JavaScript modules create their own scope, meaning variables and functions declared in a module are not automatically available globally. Each module can export specific values, functions, classes, or objects that other modules can then import. The module system supports both static imports (resolved at compile time) and dynamic imports (resolved at runtime). Module paths can be relative (`./file.js`), absolute (`/src/file.js`), or reference installed packages (`lodash`). Modern JavaScript environments and bundlers like Webpack, Vite, and Rollup handle module resolution, dependency tracking, and bundling for production deployment.

# **Related Concepts:**

---
**CommonJS (require/module.exports)**: The older Node.js module system that uses `require` for imports and `module.exports` for exports, differing from ES6 modules in being synchronous and dynamically resolved.

**[[JavaScript Module Bundlers (e.g., Webpack, Parcel, Rollup) - Overview]]**: Tools like Webpack, Rollup, and [[React - Vite]] that process modules, resolve dependencies, and bundle them into optimized files for production deployment.

**Tree Shaking**: An optimization technique used by bundlers to eliminate unused code from modules, only including the parts of modules that are actually imported and used.

**Package Management (npm/yarn)**: Systems for managing external module dependencies, where modules can be installed from registries and imported into applications.

**Scope and Closures**: JavaScript's scoping rules that modules leverage to create encapsulated environments, preventing variable leakage into the global scope.

**Asynchronous Module Loading**: Dynamic imports that return promises, allowing modules to be loaded on-demand rather than at application startup.

**Module Resolution Algorithm**: The process by which JavaScript engines and bundlers locate and load modules based on import paths, following specific rules for relative, absolute, and package imports.

# **Examples:**

---
```javascript
// math-utils.js - A module with multiple utility functions
// This demonstrates named exports

// Alternative syntax - export multiple items at once
// export { add, multiply, PI, Calculator };

// Named export - can have multiple per module
export function add(a, b) {
 return a + b;
}

// Another named export
export function multiply(a, b) {
 return a * b;
}

// Named export of a constant
export const PI = 3.14159;

// Named export of a class
export class Calculator {
 constructor {
 this.result = 0;
 }

 add(value) {
 this.result += value;
 return this;
 }

 getResult {
 return this.result;
 }
}
```

```javascript
// user-service.js - A module with a default export
// This demonstrates default exports and mixing with named exports

// Internal helper function - not exported, stays private to this module
function validateEmail(email) {
 return email.includes('@');
}

// Named export for configuration
export const API_BASE_URL = '.example.com';

// Default export - only one per module allowed
// This is the main thing this module provides
export default class UserService {
 constructor(apiUrl = API_BASE_URL) {
 this.apiUrl = apiUrl;
 }

 async createUser(userData) {
 // validateEmail is available because it's in the same module scope
 if (!validateEmail(userData.email)) {
 throw new Error('Invalid email format');
 }

 // Simulate API call
 const response = await fetch(`${this.apiUrl}/users`, {
 method: 'POST',
 headers: { 'Content-Type': 'application/json' },
 body: JSON.stringify(userData)
 });

 return response.json;
 }

 async getUser(userId) {
 const response = await fetch(`${this.apiUrl}/users/${userId}`);
 return response.json;
 }
}
```

```javascript
// app.js - Main application file demonstrating different import patterns

// Import default export - no curly braces needed
// The name 'UserService' can be anything, it refers to the default export
import UserService from './user-service.js';

// Import named exports - curly braces required, names must match exactly
import { add, multiply, PI, Calculator } from './math-utils.js';

// Import both default and named exports from the same module
import UserServiceClass, { API_BASE_URL } from './user-service.js';

// Import with alias - useful for avoiding naming conflicts
import { add as mathAdd, PI as MathPI } from './math-utils.js';

// Import everything from a module as a namespace object
import * as MathUtils from './math-utils.js';

// Dynamic import - loads module asynchronously at runtime
// Returns a Promise that resolves to the module object
async function loadUserModule {
 try {
 // This loads the module only when this function is called
 const userModule = await import('./user-service.js');
 const UserService = userModule.default; // Access default export
 const { API_BASE_URL } = userModule; // Access named export

 return new UserService;
 } catch (error) {
 console.error('Failed to load user module:', error);
 }
}

// Using imported modules
async function main {
 // Use imported functions directly
 console.log('Addition:', add(5, 3)); // Uses named import
 console.log('Multiplication:', multiply(4, 7)); // Uses named import
 console.log('PI value:', PI); // Uses named import

 // Use aliased imports
 console.log('Math add:', mathAdd(10, 20)); // Uses aliased import

 // Use namespace import
 console.log('Namespace PI:', MathUtils.PI); // Access through namespace
 console.log('Namespace calc:', MathUtils.add(1, 2)); // Access through namespace

 // Use imported class
 const calc = new Calculator;
 console.log('Calculator result:', calc.add(5).add(10).getResult);

 // Use default import (class)
 const userService = new UserService;
 try {
 const user = await userService.createUser({
 name: 'John Doe',
 email: 'john@example.com'
 });
 console.log('Created user:', user);
 } catch (error) {
 console.error('User creation failed:', error.message);
 }

 // Use dynamic import
 const dynamicUserService = await loadUserModule;
 if (dynamicUserService) {
 console.log('Dynamic module loaded successfully');
 }
}

// Run the application
main.catch(console.error);
```

```javascript
// package-imports.js - Importing external packages and handling errors

// Import from installed npm packages
// No relative path needed - bundler/Node.js resolves from node_modules
import axios from 'axios'; // Default export from axios package
import { format } from 'date-fns'; // Named export from date-fns package
import _ from 'lodash'; // Default export, commonly aliased as _

// Import from scoped packages (organizations)
import { Button } from '@mui/material'; // Named import from Material-UI

// Import CSS modules (in bundled environments)
import styles from './component.module.css'; // Default export of CSS classes object

// Handling import errors and module resolution issues
try {
 // This will work if the module exists
 const moduleExists = await import('./existing-module.js');
 console.log('Module loaded:', moduleExists);
} catch (error) {
 // Common error types:
 // 1. "Module not found: Error: Can't resolve" - file doesn't exist
 // 2. "SyntaxError" - module has syntax errors
 // 3. "TypeError" - trying to import something that wasn't exported

 if (error.message.includes("Can't resolve")) {
 console.error('Module file not found - check the file path');
 } else if (error instanceof SyntaxError) {
 console.error('Module contains syntax errors');
 } else {
 console.error('Other import error:', error.message);
 }
}

// Example of common import path issues and solutions
// ❌ Wrong - missing file extension in some environments
// import utils from './utils';

// ✅ Correct - include file extension
// import utils from './utils.js';

// ❌ Wrong - incorrect relative path
// import component from 'component.js';

// ✅ Correct - proper relative path
// import component from './component.js';

// ❌ Wrong - trying to import something not exported
// import { nonExistentFunction } from './math-utils.js';

// ✅ Correct - import only what's actually exported
// import { add } from './math-utils.js';
```

# **Flashcards:**

---
What is the difference between default exports and named exports in JavaScript modules?;; Default exports allow one primary export per module imported without curly braces (import MyComponent from './file'), while named exports allow multiple exports per module that must be imported with curly braces and exact names (import { functionName } from './file')

What does the error "Module not found: Error: Can't resolve" typically indicate?;; This error occurs when an import statement references a file path that doesn't exist, usually due to incorrect file paths, missing file extensions, wrong relative/absolute path usage, or the target file being deleted or moved

How do you import both default and named exports from the same module?;; Use the syntax: import DefaultExport, { namedExport1, namedExport2 } from './module-file.js' - the default import comes first without curly braces, followed by named imports in curly braces