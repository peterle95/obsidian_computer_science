---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date: ""
last-reviewed: 2025-08-11
scheda: done
visit-count: 4
confidence-level: 2
consecutive-correct: 1
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

# **Core Explanation:**

---
By specification, only two primitive types may serve as object property keys:

- string type, or
- symbol type.

Otherwise, if one uses another type, such as number, it’s autoconverted to string. So that `obj` is the same as `obj["1"]`, and `obj[true]` is the same as `obj["true"]`.

In JavaScript, a `Symbol` is a primitive data type introduced in ECMAScript 6 (ES6). It represents a unique and immutable value, primarily ==used as an identifier for object properties to avoid name collisions. Every time you create a Symbol, it is guaranteed to be unique, even if you provide the same description.==

**Key characteristics of JavaScript `Symbol`:**

- **Primitive Value:** `Symbol` is one of the seven primitive data types.
- **Uniqueness:** ==Every `Symbol` call creates a new, unique value. This is its most important feature.==
- **Immutability:** Once created, a Symbol's value cannot be changed.
- **Not Enumerable by `for...in` or `Object.keys`:** By default, properties whose keys are Symbols are <mark style="background:

# FF5582A6;">not enumerable</mark> when using `for...in` loops or `Object.keys`, `Object.values`, or `Object.entries`. This makes them suitable for "private-ish" object properties that are not intended for public iteration.
- **No Literal Syntax:** Symbols cannot be created using literal syntax (like `""` for strings or `123` for numbers). They must be created using the `Symbol` function.
- **Optional Description:** You can provide a string description to the `Symbol` function, which is useful for debugging but does not affect the Symbol's uniqueness.

# **Related Concepts:**

---
- **`BigInt`:** Another primitive data type introduced in ES11 for arbitrary-precision integers, distinct from `Symbol`.
- **Object Properties:** Symbols are most commonly used as keys for object properties.
- **Property Descriptors:** Symbols can be used as keys when defining properties with `Object.defineProperty`.
- **`for...in` loop:** Does not iterate over Symbol-keyed properties.
- **`Object.keys`, `Object.values`, `Object.entries`:** These methods also do not return Symbol-keyed properties.
- **`Object.getOwnPropertySymbols`:** A specific method to retrieve an array of all Symbol-keyed properties of an object.
- **`Reflect.ownKeys`:** Returns an array of all own property keys (both string and Symbol) of an object.
- **Well-known Symbols:** Built-in `Symbol` values (e.g., `Symbol.iterator`, `Symbol.toStringTag`) that are used by JavaScript's internal mechanisms to define or override standard language behavior.

# **Examples:**

---
```Javascript
//
---
Creating Symbols
---
// Creating a basic unique Symbol
const mySymbol1 = Symbol;
const mySymbol2 = Symbol;

console.log(mySymbol1); // Output: Symbol
console.log(mySymbol2); // Output: Symbol

// Symbols are unique, even with the same description
const symbolWithDesc1 = Symbol('myDescription');
const symbolWithDesc2 = Symbol('myDescription');

console.log(symbolWithDesc1); // Output: Symbol(myDescription)
console.log(symbolWithDesc2); // Output: Symbol(myDescription)

console.log(mySymbol1 === mySymbol2); // Output: false (They are unique)
console.log(symbolWithDesc1 === symbolWithDesc2); // Output: false (They are unique)

//
---

Using Symbols as Object Property Keys
---
const userId = Symbol('userId');
const userRole = Symbol('userRole'); // Another unique symbol

let user = {
 name: "Alice",
 age: 30,
 [userId]: 'a1b2c3d4e5', // Using Symbol as a property key
 [userRole]: 'admin'
};

console.log(user.name); // Output: Alice
console.log(user[userId]); // Output: a1b2c3d4e5 (Accessing Symbol-keyed property)
console.log(user[userRole]); // Output: admin

// Trying to access with a string that looks the same will not work
console.log(user['userId']); // Output: undefined

//
---

Immutability and Type
---
console.log(typeof mySymbol1); // Output: symbol

// Symbols cannot be concatenated directly with strings without explicit conversion
// console.log("User ID: " + userId); // Throws TypeError: Cannot convert a Symbol value to a string
console.log("User ID: " + String(userId)); // Output: User ID: Symbol(userId)
console.log(`User ID: ${userId.toString}`); // Output: User ID: Symbol(userId)

//
---
Symbols and Object Iteration / Reflection
---
// Symbol-keyed properties are NOT enumerable by standard methods
for (let key in user) {
 console.log(`For...in key: ${key}`); // Output: For...in key: name, age (userId and userRole are skipped)
}

console.log(Object.keys(user)); // Output: [ 'name', 'age' ]
console.log(Object.getOwnPropertyNames(user)); // Output: [ 'name', 'age' ]

// Use Object.getOwnPropertySymbols to get Symbol properties
console.log(Object.getOwnPropertySymbols(user)); // Output: [ Symbol(userId), Symbol(userRole) ]

// Reflect.ownKeys gets both string and Symbol properties
console.log(Reflect.ownKeys(user)); // Output: [ 'name', 'age', Symbol(userId), Symbol(userRole) ]

//
---
Global Symbol Registry (Symbol.for and Symbol.keyFor)
---
// Allows you to create or retrieve Symbols from a global registry
const globalSymbol1 = Symbol.for('sharedKey'); // Creates if not exists, otherwise retrieves
const globalSymbol2 = Symbol.for('sharedKey'); // Retrieves the same Symbol

console.log(globalSymbol1 === globalSymbol2); // Output: true (They are the same Symbol from the registry)

// Get the description/key of a Symbol from the global registry
console.log(Symbol.keyFor(globalSymbol1)); // Output: sharedKey

// For a Symbol not in the registry, Symbol.keyFor returns undefined
console.log(Symbol.keyFor(mySymbol1)); // Output: undefined (mySymbol1 was not created with Symbol.for)

//
---

Well-known Symbols (brief example)
---
// Used internally by JavaScript for specific behaviors
class MyIterable {
 *Symbol.iterator { // Makes MyIterable instances iterable using for...of
 yield 1;
 yield 2;
 yield 3;
 }
}
const myIterable = new MyIterable;
for (const val of myIterable) {
 console.log(val);
}
// Output: 1, 2, 3
```

# **Flashcards:**

---
What is the primary characteristic that distinguishes a `Symbol` from other JavaScript primitive data types?;; A `Symbol` is guaranteed to be a **unique** and immutable value, even if created with the same description.

How are `Symbol`-keyed properties different from string-keyed properties when iterating over an object?;; `Symbol`-keyed properties are **not enumerable** by default when using `for...in` loops, `Object.keys`, `Object.values`, or `Object.entries`. You must use `Object.getOwnPropertySymbols` or `Reflect.ownKeys`.

When would you use `Symbol.for` instead of `Symbol`?;; `Symbol.for` is used when you want to create or retrieve a **globally shared Symbol** from the JavaScript runtime's global Symbol registry, ensuring that subsequent calls with the same key return the exact same Symbol. `Symbol` always creates a brand new, unique Symbol.