---
memory: to_finish
tags:
  - to_learn
language:
  - JavaScript
review-date: 2025-11-20
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

The global object solves the problem of **providing universal access to built-in language features and environment-specific functionality**. It serves as a central namespace that makes essential variables, functions, and APIs available throughout the entire application, regardless of scope.

This is important because:
- It provides a **standardized way to access platform-specific features** (browser APIs, Node.js APIs, etc.)
- It enables **polyfilling and feature detection** for cross-browser compatibility
- It offers a **common reference point** across different JavaScript environments
- It bridges the gap between language-level features (like `Array`, `Promise`) and environment-specific features (like `window.innerHeight` in browsers)

# **Core Explanation:**
---
The global object provides variables and functions that are available anywhere. By default, those that are built into the language or the environment.

In a browser it is named `window`, for Node.js it is `global`, for other environments it may have another name.

Recently, `globalThis` was added to the language, as a standardized name for a global object, that should be supported across all environments. It’s supported in all major browsers.

We’ll use `window` here, assuming that our environment is a browser. If your script may run in other environments, it’s better to use `globalThis` instead.

All properties of the global object can be accessed directly:

```javascript
alert("Hello");
// is the same as
window.alert("Hello");
```

In a browser, global functions and variables declared with `var` (not `let/const`!) become the property of the global object:


```javascript
var gVar = 5;

alert(window.gVar); // 5 (became a property of the global object)
```

Function declarations have the same effect (statements with `function` keyword in the main code flow, not function expressions).

Please don’t rely on that! This behavior exists for compatibility reasons. Modern scripts use [JavaScript modules](https://javascript.info/modules) where such a thing doesn’t happen.

If we used `let` instead, such thing wouldn’t happen:

```javascript
let gLet = 5;

alert(window.gLet); // undefined (doesn't become a property of the global object)
```

If a value is so important that you’d like to make it available globally, write it directly as a property:

```javascript
// make current user information global, to let all scripts access it
window.currentUser = {
  name: "John"
};

// somewhere else in code
alert(currentUser.name);  // John

// or, if we have a local variable with the name "currentUser"
// get it from window explicitly (safe!)
alert(window.currentUser.name); // John
```

That said, using global variables is generally discouraged. There should be as few global variables as possible. The code design where a function gets “input” variables and produces certain “outcome” is clearer, less prone to errors and easier to test than if it uses outer or global variables.

## Using for polyfills ([[JavaScript Transpilers (e.g., Babel) and Polyfills - Overview]])

We use the global object to test for support of modern language features.

For instance, test if a built-in `Promise` object exists (it doesn’t in really old browsers):


```javascript
if (!window.Promise) {
  alert("Your browser is really old!");
}
```

If there’s none (say, we’re in an old browser), we can create “polyfills”: add functions that are not supported by the environment, but exist in the modern standard.

```javascript
if (!window.Promise) {
  window.Promise = ... // custom implementation of the modern language feature
}
```

## Summary

- The global object holds variables that should be available everywhere.
    
    That includes JavaScript built-ins, such as `Array` and environment-specific values, such as `window.innerHeight` – the window height in the browser.
    
- The global object has a universal name `globalThis`.
    
    …But more often is referred by “old-school” environment-specific names, such as `window` (browser) and `global` (Node.js).
    
- We should store values in the global object only if they’re truly global for our project. And keep their number at minimum.
    
- In-browser, unless we’re using [modules](https://javascript.info/modules), global functions and variables declared with `var` become a property of the global object.
    
- To make our code future-proof and easier to understand, we should access properties of the global object directly, as `window.x`.

# **Related Concepts:**
---

1. **Scope and Closures:**
   - The global object represents the outermost scope in JavaScript
   - All code has access to the global scope, but nested scopes can shadow global properties
   - Closures can capture variables without polluting the global namespace

2. **Variable Declarations (`var`, `let`, `const`):**
   - `var` creates properties on the global object when used at the top level
   - `let` and `const` do not create global properties, providing better encapsulation
   - This difference affects how variables interact with the global object

3. **Modules (ES6):**
   - Module scope is separate from global scope
   - Variables in modules don't automatically become global
   - Modules encourage explicit imports/exports instead of global dependencies

4. **Polyfills:**
   - Techniques to add missing features to older environments
   - Often implemented by adding properties to the global object
   - Use feature detection via the global object (e.g., `if (!window.Promise)`)

5. **Namespace Pattern:**
   - A design pattern to avoid global namespace pollution
   - Creates a single global object containing all application code
   - Example: `MyApp.utils.formatDate()` instead of just `formatDate()`

6. **`this` in Global Context:**
   - In non-strict mode, `this` refers to the global object in global scope
   - In strict mode, `this` is `undefined` in global scope
   - Related to but distinct from explicitly accessing the global object

# **Examples:**
---


```javascript
// ============================================
// Example 1: Accessing the Global Object
// ============================================

// Using the universal globalThis (recommended for cross-platform code)
console.log(globalThis); // Works in browser, Node.js, and other environments

// Browser-specific access
// console.log(window); // Only works in browser environments

// Node.js-specific access
// console.log(global); // Only works in Node.js environments


// ============================================
// Example 2: Direct vs Explicit Access
// ============================================

// Direct access to global function
alert("Hello"); // 'alert' is found on the global object (window in browsers)

// Explicit access through window object
window.alert("Hello"); // Same as above, but explicitly shows where 'alert' comes from

// This is useful to avoid conflicts with local variables
function example() {
  var alert = "I'm a local variable!"; // Shadows the global alert function
  
  // alert("Hello"); // This would cause an error - trying to call a string
  
  window.alert("Hello"); // This works - explicitly accesses global alert function
}


// ============================================
// Example 3: var vs let/const with Global Object
// ============================================

// Using var at the top level creates a global object property
var globalVar = "I'm global!";
console.log(window.globalVar); // "I'm global!" (becomes a window property)

// Using let does NOT create a global object property
let globalLet = "I'm also global, but not on window!";
console.log(window.globalLet); // undefined (does not become a window property)

// Using const behaves like let
const globalConst = "Same as let!";
console.log(window.globalConst); // undefined (does not become a window property)

// Function declarations also become global properties
function globalFunction() {
  return "I'm accessible via window!";
}
console.log(window.globalFunction); // Function is accessible as window property
console.log(window.globalFunction()); // "I'm accessible via window!"


// ============================================
// Example 4: Explicitly Setting Global Properties
// ============================================

// Instead of relying on var, explicitly set global properties
// This makes the intention clear and works consistently
window.appConfig = {
  apiUrl: "https://api.example.com",
  version: "1.0.0"
};

// Now accessible anywhere in the application
console.log(appConfig.apiUrl); // Direct access
console.log(window.appConfig.apiUrl); // Explicit access (safer)

// If there's a local variable with same name, window access prevents conflicts
function checkConfig() {
  var appConfig = "local value"; // Local variable shadows global
  
  console.log(appConfig); // "local value" (local variable)
  console.log(window.appConfig); // { apiUrl: "...", version: "..." } (global object)
}


// ============================================
// Example 5: Feature Detection and Polyfills
// ============================================

// Check if a modern feature exists in the environment
if (!window.Promise) {
  // Browser doesn't support Promises - alert the user
  alert("Your browser is really old!");
  
  // In a real application, you would add a polyfill here
  window.Promise = function(executor) {
    // Custom Promise implementation for old browsers
    // (simplified - real polyfills are more complex)
  };
}

// Check for other modern features
if (!window.fetch) {
  console.log("Fetch API not supported, loading polyfill...");
  // Load fetch polyfill
}

// Feature detection for array methods
if (!Array.prototype.includes) {
  // Add the missing method to Array prototype
  Array.prototype.includes = function(searchElement) {
    // Polyfill implementation
    return this.indexOf(searchElement) !== -1;
  };
}


// ============================================
// Example 6: Best Practices - Avoiding Global Pollution
// ============================================

// ❌ BAD: Multiple global variables pollute namespace
var userName = "John";
var userAge = 30;
var userEmail = "john@example.com";

// ✅ GOOD: Single global namespace object
window.MyApp = {
  user: {
    name: "John",
    age: 30,
    email: "john@example.com"
  },
  config: {
    theme: "dark"
  },
  utils: {
    formatDate: function(date) {
      return date.toLocaleDateString();
    }
  }
};

// Access through the namespace
console.log(MyApp.user.name); // "John"
console.log(MyApp.utils.formatDate(new Date())); // Formatted date


// ============================================
// Example 7: Module Pattern to Avoid Globals
// ============================================

// Instead of using global variables, use modules (when available)
// Modern approach with ES6 modules:

// In userModule.js:
// export const userName = "John";
// export function greetUser() { return `Hello, ${userName}`; }

// In main.js:
// import { userName, greetUser } from './userModule.js';
// console.log(greetUser()); // "Hello, John"

// This keeps code out of global scope entirely
```

# **Flashcards:**
---


What is the global object in JavaScript and what is its primary purpose?;; The global object is a special object that provides variables and functions available anywhere in the code. It serves as the root namespace containing built-in JavaScript features (like Array, Promise) and environment-specific values. In browsers it's `window`, in Node.js it's `global`, and the universal standardized name is `globalThis`.

What is the difference between how `var` and `let`/`const` declarations interact with the global object?;; Variables declared with `var` at the top level become properties of the global object (e.g., `var x = 5` makes `window.x` equal to 5). In contrast, `let` and `const` declarations do NOT become properties of the global object, even when declared at the top level. This provides better encapsulation in modern JavaScript.

What is `globalThis` and why was it introduced?;; `globalThis` is a standardized name for the global object that works across all JavaScript environments (browsers, Node.js, Web Workers, etc.). It was introduced to provide a universal way to access the global object without needing to use environment-specific names like `window` or `global`, making code more portable.

How can you use the global object for feature detection and polyfills?;; You can check if a feature exists by testing for its presence on the global object: `if (!window.Promise) { /* feature missing */ }`. If a feature is missing, you can add a polyfill by assigning your implementation to the global object: `window.Promise = /* custom implementation */`. This allows older browsers to support modern features.

Why is it recommended to minimize the use of global variables?;; Global variables can lead to several problems: namespace pollution (conflicts with other code), harder debugging (unclear where values come from), tight coupling (code depends on global state), and difficulty testing (functions relying on globals need specific global setup). Code with explicit inputs/outputs is clearer, less error-prone, and easier to test.

How should you explicitly access global properties to avoid conflicts with local variables?;; Use the global object name as a prefix: `window.propertyName` in browsers or `globalThis.propertyName` universally. For example, if you have a local variable `var alert = "text"` that shadows the global `alert()` function, you can still access the function with `window.alert()`. This explicit access makes code safer and intentions clearer.
