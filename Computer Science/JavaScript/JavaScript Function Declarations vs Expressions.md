---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date: ""
last-reviewed: 2025-07-22
scheda: done
visit-count: 3
confidence-level: 1
consecutive-correct: 1
last-struggle-date: 2025-07-12

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
Understanding the difference between Function Declarations and Function Expressions in JavaScript is crucial because it directly impacts ==**when and where a function can be accessed**== within your code, primarily due to a concept called **[[JavaScript Hoisting]]**. This distinction is vital for avoiding unexpected errors, writing predictable and robust code, and leveraging the specific behaviors of each definition style. It influences code organization, debugging, and the effective use of JavaScript's lexical environment.

# **Core Explanation:**

---

In JavaScript, there are two primary ways to define a named function: as a **Function Declaration** or as a **Function Expression**. While both create callable functions, they differ significantly in their parsing and availability due to JavaScript's hoisting mechanism.

1. **Function Declaration**:
 * **Syntax**: `function functionName(parameters) { /* code */ }`
 * **Hoisting**: ==Function Declarations are **hoisted**==. This means that the JavaScript interpreter moves their definitions to the top of their containing scope *before* the code actually executes. Consequently, you can call a function declared this way *before* its actual definition appears in the code.
 * **Characteristics**:
 * Must have a name.
 * Available throughout the entire scope in which they are declared (e.g., global or within another function).
 * Often preferred for general-purpose functions that need to be accessible throughout a script or block.

2. **Function Expression**:
 * **Syntax**: `const functionName = function(parameters) { /* code */ };` or `let functionName = function(parameters) { /* code */ };`
 * Can also be an **Arrow Function** (a more concise form introduced in ES6): `const functionName = (parameters) => { /* code */ };`
 * **Hoisting**: ==Function Expressions are **not hoisted** in the same way. Only the variable name (e.g., `functionName`) is hoisted, but its *assignment* to the function definition happens only when that line of code is executed. Therefore, you **cannot call a function expression before the line where it is defined*==*.
 * **Characteristics**:
 * Can be **anonymous** (no name after `function` keyword, e.g., `const func = function {}`).
 * Can be **named** (e.g., `const func = function myFunc {}`; `myFunc` is only available inside the function's own scope).
 * Typically used when a function is assigned as a value to a variable, passed as an argument to another function (callbacks), or defined conditionally.
 * Their availability is sequential; they are only callable after their definition line has been parsed and executed.

# **Related Concepts:**

---
* **Hoisting**: This is the core concept distinguishing declarations from expressions. Hoisting is JavaScript's default behavior of moving declarations to the top of the current scope (global or function scope) before code execution. While function declarations are fully hoisted (both declaration and definition), only the variable declaration part of a function expression is hoisted, not its assignment to the function value.
* **Scope**: Both function declarations and expressions are bound by JavaScript's lexical scope. Where they are defined (global scope, inside another function, or within a block) determines their accessibility. Variables declared within a function (either type) are local to that function's scope.
* **`var`, `let`, `const`**: When defining function expressions, the choice of `var`, `let`, or `const` influences their variable hoisting and reassignability. `const` is generally preferred for function expressions to prevent accidental reassignment. `var` variables are function-scoped and hoisted, while `let` and `const` are block-scoped and exhibit a "temporal dead zone" where they are not accessible before their declaration.
* **Arrow Functions**: Introduced in ES6, arrow functions are a more concise syntax for writing function expressions. They have lexical `this` binding, which behaves differently from traditional function declarations/expressions, making them particularly useful for callbacks.
* **Callback Functions**: Function expressions are very commonly used as callback functions, where a function is passed as an argument to another function to be executed later. This pattern relies on the function expression being defined and available when the outer function is called.

# **Examples:**

---
```javascript
//
---
Function Declaration Example
---
// A function declaration can be called before its definition because of hoisting.
sayHelloDeclaration; // This works! Output: Hello from Declaration!

function sayHelloDeclaration {
 console.log("Hello from Declaration!");
}

// You can, of course, also call it after its definition.
sayHelloDeclaration; // Output: Hello from Declaration!

//
---

Function Expression Example
---
// Attempting to call a function expression before its definition will result in an error.
// sayHelloExpression; // This line would cause a ReferenceError: Cannot access 'sayHelloExpression' before initialization

// Define a function as an expression, assigning it to a constant variable.
const sayHelloExpression = function {
 console.log("Hello from Expression!");
};

// Now, after its definition, the function expression can be called.
sayHelloExpression; // Output: Hello from Expression!

//
---

Function Expression with a Name (Named Function Expression)
---
// The name 'internalName' is only accessible inside the function itself.
const calculateSum = function internalName(a, b) {
 // console.log(internalName); // 'internalName' is accessible here
 return a + b;
};

console.log(calculateSum(10, 20)); // Output: 30
// console.log(internalName); // This would cause a ReferenceError, 'internalName' is not accessible outside.

//
---

Arrow Function Example (a type of Function Expression)
---
// Define an arrow function (a concise function expression)
const greetArrow = (name) => {
 console.log(`Greetings, ${name} from Arrow Function!`);
};

// Call the arrow function
greetArrow("Charlie"); // Output: Greetings, Charlie from Arrow Function!

//
---
Conditional Function Definition (Common use case for Function Expressions)
---
let canGreet = true;
let conditionalGreeting;

if (canGreet) {
 // Function expression defined conditionally based on a condition
 conditionalGreeting = function {
 console.log("Conditional greeting activated!");
 };
} else {
 conditionalGreeting = function {
 console.log("Conditional greeting skipped.");
 };
}

// Call the conditionally defined function
conditionalGreeting; // Output: Conditional greeting activated!

// If canGreet was false, it would output "Conditional greeting skipped."
````

# **Flashcards:**

---
What is the key difference in hoisting between Function Declarations and Function Expressions?;;Function Declarations are fully hoisted (callable before definition); Function Expressions are not (only the variable is hoisted, function is callable only after its definition).

When can you call a Function Declaration?;;Anywhere within its scope, even before its actual definition in the code.

When can you call a Function Expression?;;Only after the line of code where it is defined has been executed.