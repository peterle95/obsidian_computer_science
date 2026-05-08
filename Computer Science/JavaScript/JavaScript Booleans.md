---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date: ""
last-reviewed: 2025-07-18
keywords:
 - boolean
 - "true"
 - "false"
 - truthy
 - falsy
 - logical operators
 - conditional statements
scheda: done
visit-count: 2
confidence-level: 2
consecutive-correct: 2

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

# **Core Explanation:**

---
In JavaScript, the `boolean` primitive data type represents a logical entity and can have only one of two values: `true` or `false`. Booleans are fundamental for controlling program flow, making decisions, and representing binary states (e.g., "is active," "has permission," "is logged in").

**Key characteristics of JavaScript Booleans:**

- **Primitive Value:** `boolean` is one of the seven primitive data types.
- **Binary State:** It exclusively represents a true or false condition.
- **Contextual Conversion (Truthy and Falsy):** While there are explicit `true` and `false` values, ==JavaScript also has a concept of "truthy" and "falsy" values==. In a boolean context (like `if` statements, `while` loops, or logical operations), non-boolean values are automatically ==coerced into a boolean equivalent==.
 >- **Falsy values** are values that JavaScript treats as `false` when encountered in a boolean context. These include: `false`, `0` (the number zero), `-0` (negative zero), `""` (empty string), `null`, `undefined`, and `NaN`.
 - **Truthy values** are all other values that are not explicitly falsy. This includes empty arrays (``), empty objects (`{}`), and even the string `"0"` or `" "` (a string with a space).
- **`Boolean` Object [[Wrapper]]:** Similar to `String` and `Number`, there's a `Boolean` object wrapper (`new Boolean`). However, directly using `Boolean` objects is rarely necessary and can lead to unexpected behavior (e.g., `new Boolean(false)` is a truthy object). Always prefer primitive boolean values.

# **Related Concepts:**

---
- **Logical Operators:**
 - **Logical AND (`&&`):** Returns `true` if both operands are truthy; otherwise, `false`. It also performs short-circuit evaluation.
 - **Logical OR (`||`):** Returns `true` if at least one operand is truthy; otherwise, `false`. It also performs short-circuit evaluation.
 - **Logical NOT (`!`):** Inverts the boolean value of its operand.
- **Conditional Statements:** `if...else if...else`, `switch` statements, and the ternary operator (`condition ? expr1 : expr2`) rely heavily on boolean evaluation.
- **Comparison Operators:** Operators that compare two values and return a boolean result (e.g., `/===` strict equality, `/==` loose equality, `>`, `<`, `>=`, `<=`).
 --> use strict equality most of the time
- **Type Coercion:** The automatic conversion of values to booleans in logical contexts is a prime example of JavaScript's type coercion.
- **Control Flow:** Booleans directly influence the execution path of a program.

# **Examples:**

---
```Javascript
//
---
Boolean Literals
---
let isAdult = true;
let hasVoted = false;

console.log(isAdult); // Output: true
console.log(typeof isAdult); // Output: boolean

console.log(hasVoted); // Output: false
console.log(typeof hasVoted); // Output: boolean

//
---

Comparison Operators (return booleans)
---
let x = 10;
let y = 20;

console.log(x < y); // Output: true
console.log(x === y); // Output: false (strict equality: checks value and type)
console.log(x == '10'); // Output: true (loose equality: allows type coercion)
console.log(x !== y); // Output: true

//
---

Logical Operators
---
let age = 25;
let hasLicense = true;

// Logical AND (&&)
// Both conditions must be true for the result to be true
let canDrive = age >= 18 && hasLicense;
console.log("Can drive?", canDrive); // Output: Can drive? true

// Logical OR (||)
// At least one condition must be true for the result to be true
let canEnterClub = age >= 18 || isAdult; // isAdult is already true
console.log("Can enter club?", canEnterClub); // Output: Can enter club? true

// Logical NOT (!)
// Inverts the boolean value
let isLoggedIn = false;
console.log("Is NOT logged in?", !isLoggedIn); // Output: Is NOT logged in? true

//
---

Truthy and Falsy Values
---
// Falsy values
console.log(Boolean(false)); // Output: false
console.log(Boolean(0)); // Output: false
console.log(Boolean(-0)); // Output: false
console.log(Boolean("")); // Output: false (empty string)
console.log(Boolean(null)); // Output: false
console.log(Boolean(undefined)); // Output: false
console.log(Boolean(NaN)); // Output: false

// Truthy values (anything not explicitly falsy)
console.log(Boolean(true)); // Output: true
console.log(Boolean(1)); // Output: true
console.log(Boolean(-100)); // Output: true
console.log(Boolean("hello")); // Output: true (non-empty string)
console.log(Boolean(" ")); // Output: true (string with space)
console.log(Boolean()); // Output: true (empty array is truthy!)
console.log(Boolean({})); // Output: true (empty object is truthy!)
console.log(Boolean(function {})); // Output: true (functions are truthy)

//
---

Using Booleans in Conditional Statements
---
let temperature = 28;

if (temperature > 25) {
 console.log("It's hot outside!");
} else {
 console.log("Temperature is moderate or cool.");
}
// Output: It's hot outside!

let user = { name: "John" }; // This object is truthy

if (user) { // The object 'user' is truthy, so this block executes
 console.log(`User ${user.name} is defined.`);
} else {
 console.log("User is undefined.");
}
// Output: User John is defined.

let email = ""; // This empty string is falsy

if (email) { // The empty string '' is falsy, so this block does NOT execute
 console.log("Email is provided.");
} else {
 console.log("Please provide an email.");
}
// Output: Please provide an email.

//
---
Short-circuiting with logical operators
---
// && (AND) short-circuits if the first operand is falsy
let result1 = false && "This will not be evaluated";
console.log(result1); // Output: false

let result2 = true && "This will be evaluated and returned";
console.log(result2); // Output: This will be evaluated and returned

// || (OR) short-circuits if the first operand is truthy
let result3 = true || "This will not be evaluated";
console.log(result3); // Output: true

let result4 = false || "This will be evaluated and returned";
console.log(result4); // Output: This will be evaluated and returned

// Common use case: assigning a default value
let userName = null;
let displayUserName = userName || "Guest";
console.log(displayUserName); // Output: Guest (because userName is falsy)

let anotherUserName = "Jane";
let anotherDisplayUserName = anotherUserName || "Guest";
console.log(anotherDisplayUserName); // Output: Jane (because anotherUserName is truthy)
```

# **Flashcards:**

---
What are the only two possible values for the JavaScript `boolean` data type?;; `true` and `false`.

Explain the concept of "truthy" and "falsy" values in JavaScript.;; "Falsy" values (like `0`, `""`, `null`, `undefined`, `NaN`, `false`) are treated as `false` in a boolean context. All other values are "truthy" (e.g., ``, `{}`, non-empty strings, any non-zero number).

Which logical operator in JavaScript returns `true` if at least one operand is truthy, and what is its symbol?;; The Logical OR operator, represented by `||`.