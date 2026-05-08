---
memory: to_finish
tags:
  - learned
language:
  - JavaScript
review-date: ""
last-reviewed: 2025-07-18
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 3
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
==JavaScript categorizes its data into two fundamental types: Primitives and Objects.== This distinction is crucial because they are stored and ==manipulated differently in memory.==

**Primitive Data Types** represent single, simple values. They are immutable, meaning their value cannot be changed after creation. When you assign a primitive variable to another, a copy of the value is made. There are seven primitive data types in JavaScript:

- **`string`**: Represents sequences of characters (e.g., `"hello"`).
- **`number`**: Represents both integer and floating-point numbers (e.g., `10`, `3.14`).
- **`boolean`**: Represents a logical entity and can only have two values: `true` or `false`.
- **`undefined`**: Represents a variable that has been declared but not yet assigned a value.
- **`null`**: Represents the intentional absence of any object value. It's often used to indicate that a variable points to no object.
- **`symbol`** (ES6): Represents a unique identifier. Symbols are often used as keys for object properties to avoid name collisions.
- **`bigint`** (ES11): Represents whole numbers larger than 253−1, which is the largest number `number` can reliably represent.

**Object Data Types** (or non-primitive data types) are collections of properties, where each property has a name (or key) and a value. Unlike primitives, objects are mutable, meaning their properties can be changed after creation. ==When you assign an object variable to another, a reference to the same object in memory is copied, not the object itself.== Common object types include:

- **`Object`**: The most fundamental object type, used for creating generic collections of key-value pairs (e.g., `{ name: "Alice", age: 30 }`).
- **`Array`**: A special type of object used for ordered collections of values (e.g., `[1, 2, 3]`).
- **`Function`**: A callable object that executes a block of code. Functions are first-class citizens in JavaScript, meaning they can be treated like any other variable.
- **`Date`**: Objects for working with dates and times.
- **`RegExp`**: Objects for working with regular expressions.

# **Related Concepts:**

---
- **Pass by Value vs. Pass by Reference**: Primitives are "passed by value" (a copy is made), while objects are "passed by reference" (a reference to the same memory location is used).
- **Mutability vs. Immutability**: Primitives are immutable; objects are mutable.
- **`typeof` operator**: Used to determine the data type of a variable or expression.
- **Type Coercion**: JavaScript's automatic conversion of values from one data type to another.
- [[JavaScript Strings]]
- [[JavaScript Numbers]]
- [[JavaScript Booleans]]
- [[JavaScript Null]]
- [[JavaScript Undefined]]
- [[JavaScript Symbols (ES6)]]
- [[JavaScript BigInt (ES11)]]

# **Examples:**

---
```Javascript
//
---
Primitive Data Types
---
// string
let greeting = "Hello, World!";
console.log(typeof greeting); // Output: string

// number
let age = 30;
let price = 19.99;
console.log(typeof age); // Output: number
console.log(typeof price); // Output: number

// boolean
let isActive = true;
let hasPermission = false;
console.log(typeof isActive); // Output: boolean
console.log(typeof hasPermission); // Output: boolean

// undefined
let userName; // Declared but not assigned
console.log(typeof userName); // Output: undefined

// null - (Note: typeof null is an historical bug, it should be 'null' but returns 'object')
let selectedItem = null;
console.log(typeof selectedItem); // Output: object (historical bug, still prevalent)
console.log(selectedItem === null); // Output: true (correct way to check for null)

// symbol
const id1 = Symbol('uniqueId');
const id2 = Symbol('uniqueId');
console.log(typeof id1); // Output: symbol
console.log(id1 === id2); // Output: false (Symbols are always unique)

// bigint
const bigNumber = 1234567890123456789012345678901234567890n;
console.log(typeof bigNumber); // Output: bigint

//
---

Object Data Types
---
// Object (generic object)
let person = {
 name: "Alice",
 age: 25,
 isStudent: false
};
console.log(typeof person); // Output: object
console.log(person.name); // Output: Alice

// Array (a special type of object)
let numbers = [1, 2, 3, 4, 5];
console.log(typeof numbers); // Output: object
console.log(Array.isArray(numbers)); // Output: true (better way to check for arrays)
console.log(numbers); // Output: 1

// Function (a callable object)
function greet(name) {
 return "Hi, " + name + "!";
}
console.log(typeof greet); // Output: function
console.log(greet("Bob")); // Output: Hi, Bob!

// Date object
const today = new Date;
console.log(typeof today); // Output: object
console.log(today.getFullYear); // Output: current year

// Example of pass by value (primitives)
let a = 10;
let b = a; // b gets a copy of a's value
b = 20;
console.log(a); // Output: 10 (a remains unchanged)
console.log(b); // Output: 20

// Example of pass by reference (objects)
let obj1 = { value: 10 };
let obj2 = obj1; // obj2 gets a reference to the same object as obj1
obj2.value = 20;
console.log(obj1.value); // Output: 20 (obj1's value is changed because obj2 references the same object)
console.log(obj2.value); // Output: 20
```

# **Flashcards:**

---
What are the two main categories of JavaScript data types?;; Primitive and Object.

Name the seven primitive data types in JavaScript.;; `string`, `number`, `boolean`, `undefined`, `null`, `symbol`, and `bigint`.

What is the key difference in how primitive and object data types are stored and manipulated in memory?;; Primitives store the actual value and are passed by value (a copy is made), while objects store a reference to their memory location and are passed by reference (the reference is copied).