---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date:
last-reviewed: 2025-08-24
scheda: done
visit-count: 5
confidence-level: 3
consecutive-correct: 5

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
.me/blog/scope-hoisting.html?utm_source=tldrwebdev

Hoisting isn't a feature to be actively used, but rather a fundamental behavior of the JavaScript engine that developers must understand. It describes how variable and function declarations are processed during the compilation (is js compiled??) phase, before the code is executed.

The primary reason it's important is for **predictability and error avoidance**. Understanding hoisting helps you know why you can call a function before it's written in the code, or why a `var` variable might return `undefined` when accessed before its assignment. A solid grasp of this concept is crucial for debugging and writing robust, error-free JavaScript code, especially when dealing with variable scope and lifecycle.

Based on your JavaScript hoisting study material, let me address your questions:

#

# When Hoisting Becomes Useful

Hoisting is primarily useful for **understanding and debugging** rather than actively leveraging:

- **Function declarations**: You can call functions before they're defined in the code, which allows for more flexible code organization
- **Debugging**: Understanding why `var` variables return `undefined` instead of throwing errors when accessed before assignment
- **Code predictability**: Knowing how the JavaScript engine processes your code helps prevent unexpected behaviors

#

# Variable Declaration vs Initialization Hoisting

- **Declarations are hoisted**: `var x;` gets moved to the top of the scope
- **Initializations are NOT hoisted**: `var x = 5;` - only the `var x;` part moves up, the `= 5` stays where you wrote it

This is why accessing a `var` before its assignment gives `undefined` rather than an error.

#

# Forward Declarations and Circular Dependencies

**You don't need forward declarations like C++!** Here's why:

- **Function declarations** are fully hoisted (both declaration and definition), so you can call them anywhere in their scope
- **Variable declarations** are hoisted but initialized as `undefined`, eliminating reference errors
- **Circular dependencies** between functions work because function declarations are completely available throughout their scope

However, be careful with `let`/`const` - they're hoisted but in a "temporal dead zone" until their declaration line, so they behave more like C++ in requiring declaration before use.

The key difference from C++ is that JavaScript's compilation phase handles the dependency resolution for you automatically.

# **Core Explanation:**

---
**Hoisting** is JavaScript's default behavior of moving all `var` and `function` declarations to the top of their containing scope (==either function scope or global scope==) during the compilation phase. It's important to note that only the **declarations** are hoisted, not the **initializations** (the assignment of a value).

* **`var` declarations**: Are hoisted and initialized with a value of `undefined`. This means you can access a `var` variable before its declaration line without an error, but its value will be `undefined` until the assignment is reached.
* **`function` declarations**: Are fully hoisted. This means both the declaration and the function body are moved to the top, allowing you to call a function before it is physically defined in the code.
* **`let` and `const` declarations**: Are also hoisted, but they are not initialized. They are placed in a [[Temporal Dead Zone (TDZ)]] from the start of the scope until the line where they are declared. Accessing them within the TDZ results in a `ReferenceError`.

# **Related Concepts:**

---
* **Execution Context**: Hoisting is a direct result of how the JavaScript engine creates an execution context. The context is created in two phases: the **Creation Phase** (where declarations are registered in memory—this is hoisting) and the **Execution Phase** (where the code is run line by line and assignments are made).
* **Temporal Dead Zone (TDZ)**: This is directly tied to the hoisting of `let` and `const`. The TDZ is the period between entering a scope and the actual declaration of a `let` or `const` variable. While technically hoisted, these variables are inaccessible in the TDZ, providing a stricter and more predictable behavior than `var`.
* **Scope**: Hoisting happens within the current scope of the variable or function (either global, function, or block scope for `let` and `const`). Understanding scope is essential to know *where* a declaration gets hoisted to.

# **Examples:**

---
var declarations are processed when the function starts (or script starts for globals).

In other words, var variables are defined from the beginning of the function, no matter where the definition is (assuming that the definition is not in the nested function).

So this code:
```javascript
function sayHi {
 phrase = "Hello";

 alert(phrase);

 var phrase;
}
sayHi;
```
…Is technically the same as this (moved var phrase above):
```javascript
function sayHi {
 var phrase;

 phrase = "Hello";

 alert(phrase);
}
sayHi;
```
…Or even as this (remember, code blocks are ignored):
```javascript
function sayHi {
 phrase = "Hello"; // (*)

 if (false) {
 var phrase;
 }

 alert(phrase);
}
sayHi;
```
People also call such behavior “hoisting” (raising), because all var are “hoisted” (raised) to the top of the function.

So in the example above, if (false) branch never executes, but that doesn’t matter. The var inside it is processed in the beginning of the function, so at the moment of (\*) the variable exists.

Declarations are hoisted, but assignments are not.

That’s best demonstrated with an example:
```javascript
function sayHi {
 alert(phrase);

 var phrase = "Hello";
}

sayHi;
```
The line var phrase = "Hello" has two actions in it:

- Variable declaration var
- Variable assignment =.

The declaration is processed at the start of function execution (“hoisted”), but the assignment always works at the place where it appears. So the code works essentially like this:
```javascript
function sayHi {
 var phrase; // declaration works at the start...

 alert(phrase); // undefined

 phrase = "Hello"; // ...assignment - when the execution reaches it.
}

sayHi;
```
Because all var declarations are processed at the function start, we can reference them at any place. But variables are undefined until the assignments.

In both examples above, alert runs without an error, because the variable phrase exists. But its value is not yet assigned, so it shows undefined.

#

#

# 2nd Example:

When `var` variables are hoisted, JavaScript only hoists ==the declaration, not the initialization==. Here's what happens behind the scenes:

```javascript
// What you write:
console.log(y); // Outputs: undefined
var y = 20;

// What JavaScript actually does during hoisting:
var y; // Declaration is hoisted to the top
console.log(y); // y exists but has no value yet, so it's undefined
y = 20; // The assignment happens here in the original position
```

The variable `y` exists when `console.log(y)` is executed because the declaration was hoisted, but its value hasn't been assigned yet. JavaScript automatically initializes hoisted `var` variables with the value `undefined` until they're actually assigned a value in the code execution.

This is different from `let` variables which also have their declarations hoisted but remain in the "temporal dead zone" (completely inaccessible) until their declaration statement is reached.

# **Flashcards:**

---
What is JavaScript hoisting?;;JavaScript's default behavior of moving `var` and `function` declarations to the top of their scope during the compilation phase, before code execution. Initializations are not hoisted.

How does hoisting affect `var` vs. `let` and `const`?;;`var` declarations are hoisted and initialized with `undefined`. `let` and `const` declarations are also hoisted but are not initialized; they are placed in a Temporal Dead Zone (TDZ) and throw a `ReferenceError` if accessed before their declaration line.

Are function expressions hoisted in JavaScript?;;No. Only function _declarations_ are fully hoisted (both name and body). Function _expressions_ are treated like variable assignments; if assigned to a `var`, the variable name is hoisted and is `undefined`, but the function body is not.