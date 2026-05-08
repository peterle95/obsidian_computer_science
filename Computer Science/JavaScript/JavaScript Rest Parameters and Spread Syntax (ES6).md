---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date:
last-reviewed: 2025-08-22
scheda: done
visit-count: 5
confidence-level: 2
consecutive-correct: 3
last-struggle-date: 2025-07-14

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
^

# **Purpose/Why**:

---
Both Rest Parameters and Spread Syntax, introduced in ES6 (ECMAScript 2015), enhance flexibility and conciseness in JavaScript.

==**Rest Parameters** solve the problem of **handling an indefinite number of arguments** passed to a function. Before rest parameters, developers relied on the less flexible and array-like `arguments` object==. Rest parameters provide a clean, modern, and array-based way to gather any remaining arguments into a single array, making functions more adaptable and easier to work with when the exact number of inputs isn't known beforehand.

**Spread Syntax** addresses the need for **easily expanding iterables (like arrays or strings) into individual elements** or for **creating shallow copies/merges of arrays and objects** without resorting to verbose methods like `concat` or `Object.assign`. It simplifies common array and object manipulation tasks, leading to more readable and efficient code for non-mutating operations.

Together, they empower developers to write more dynamic, functional, and cleaner JavaScript code.

# **Core Explanation:**

---
#

#

# **Rest Parameters (`...`)**
Rest parameters allow a function to accept an indefinite number of arguments as an array. They are denoted by three dots (`...`) followed by a parameter name in the function's definition.

* **Syntax**: ==`function func(...args) { ... }`==
* **Behavior**:
 * Gathers all remaining arguments (that haven't been mapped to preceding named parameters) into a **real JavaScript `Array`**.
 * ==Must be the **last parameter** in the function definition.== A function can have only <mark style="background:

# BBFABBA6;">one rest parameter.</mark>
* **Purpose**: Useful when you want a function to be able to take any number of inputs for a specific purpose (e.g., a `sum` function that can add any number of numbers). It's a modern replacement for the older `arguments` object, offering true array methods.

#

#

# **Spread Syntax (`...`)**
Spread syntax allows an iterable (like an array or a string) to be expanded into individual elements. It uses the same three dots (`...`) but is used in different contexts outside of a function's parameter list.

* **Syntax**: ==`...iterable`==
* **Behavior**:
 >* **In Array Literals**: Expands an array into a list of its elements, useful for creating new arrays, concatenating, or shallow copying: `[...arr1, ...arr2]`.
 >* **In Function Calls**: Expands an array (or other iterable) into individual arguments that are passed to a function: `func(...myArray)`.
 >* **In Object Literals (ES2018)**: Copies enumerable properties from one or more source objects into a new object: `{ ...obj1, ...obj2 }`. Useful for merging objects or shallow copying.
* **Purpose**: Simplifies array and object manipulation, making operations like copying, merging, and passing dynamic arguments much more concise and readable.

==**Key Difference**: While both use `...`, **Rest Parameters *gather* elements into an array**, and **Spread Syntax *expands* an iterable into individual elements**.==

# **Related Concepts:**

---
* **[[JavaScript Function Parameters and Arguments]]**: Rest parameters directly extend the concept of function parameters, providing a way to handle a variable number of arguments dynamically.
* **Arrays**: Rest parameters *become* arrays, and Spread syntax *operates on* arrays (and other iterables) by expanding them. A deep understanding of array methods and properties is beneficial for both.
* **Objects**: Spread syntax extends to objects (ES2018), allowing for concise copying and merging of object properties, similar to how it works with arrays.
* **Iterables**: Spread syntax is designed to work with any iterable object (like strings, `Set`, `Map`), meaning it can expand their elements.
* **`arguments` object**: This is the pre-ES6 equivalent of Rest parameters. The `arguments` object is an array-like (but not true array) object available inside functions, containing all arguments passed. Rest parameters are generally preferred as they result in a real array and are more explicit.
* **Shallow Copy vs. Deep Copy**: Both array and object spread syntax perform *shallow* copies. This means nested objects or arrays within the copied structure are still references to the original nested structures, not independent copies.

# **Examples:**

---
```javascript
//
---
Rest Parameters Examples
---
// 1. Gathering an indefinite number of arguments
function sumAll(...numbers) {
 // 'numbers' will be an array containing all arguments passed to this function
 return numbers.reduce((total, num) => total + num, 0);
}

console.log(sumAll(1, 2, 3)); // Output: 6 (numbers = [1, 2, 3])
console.log(sumAll(10, 20, 30, 40)); // Output: 100 (numbers = [10, 20, 30, 40])
console.log(sumAll); // Output: 0 (numbers = )

// 2. Combining with regular parameters
function greet(greeting, ...names) {
 // 'greeting' gets the first argument, 'names' gets the rest as an array
 if (names.length === 0) {
 return `${greeting}!`;
 }
 return `${greeting}, ${names.join(" and ")}!`;
}

console.log(greet("Hello", "Alice", "Bob")); // Output: Hello, Alice and Bob!
console.log(greet("Hi")); // Output: Hi!
console.log(greet("Hey", "Charlie")); // Output: Hey, Charlie!

// 3. Rest parameter must be the last
// function invalidFunc(...args, lastParam) {} // This would cause a SyntaxError!

//
---
Spread Syntax Examples
---
// 1. Expanding an array into function arguments
const nums = [1, 2, 3];
console.log(Math.max(nums)); // Output: NaN (Math.max expects individual numbers, not an array)
console.log(Math.max(...nums)); // Output: 3 (Expands [1, 2, 3] into 1, 2, 3)

const otherNums = [10, 20];
console.log(sumAll(...otherNums, 5)); // Output: 35 (Expands [10, 20] into 10, 20, then sums with 5)

// 2. Creating new arrays (shallow copy)
const originalArray = [1, 2, 3];
const copiedArray = [...originalArray]; // Creates a new array with elements from originalArray
console.log(copiedArray); // Output: [1, 2, 3]
console.log(originalArray === copiedArray); // Output: false (They are different arrays in memory)

originalArray = 99;
console.log(originalArray); // Output: [99, 2, 3]
console.log(copiedArray); // Output: [1, 2, 3] (Copy was made before modification)

// 3. Concatenating arrays
const arr1 = [1, 2];
const arr2 = [3, 4];
const combinedArray = [...arr1, ...arr2, 5]; // Concatenates and adds new elements
console.log(combinedArray); // Output: [1, 2, 3, 4, 5]

// 4. Adding elements to an array
const baseFruits = ["apple", "banana"];
const newFruits = ["orange", ...baseFruits, "grape"];
console.log(newFruits); // Output: ["orange", "apple", "banana", "grape"]

// 5. Spreading objects (ES2018) - shallow copy/merge
const userProfile = { name: "John", age: 30 };
const updatedProfile = { ...userProfile, city: "New York" }; // Copy and add a new property
console.log(updatedProfile); // Output: { name: "John", age: 30, city: "New York" }

const adminUser = { role: "admin", ...userProfile }; // Merge, order matters for conflicts
console.log(adminUser); // Output: { role: "admin", name: "John", age: 30 }

const conflictingProps = { name: "Jane", age: 25 };
const mergedProfile = { ...userProfile, ...conflictingProps }; // 'conflictingProps' overrides 'userProfile' for same keys
console.log(mergedProfile); // Output: { name: "Jane", age: 25 }

// Note on shallow copy with nested objects:
const userWithAddress = {
 name: "Leo",
 address: { street: "Main St", zip: "10001" }
};
const copiedUser = { ...userWithAddress };
copiedUser.address.street = "Elm St"; // Modifying nested object in copiedUser also affects userWithAddress
console.log(userWithAddress.address.street); // Output: Elm St
````

# **Flashcards:**

---
What is the syntax for Rest Parameters, and what do they do?;;...parameterName in a function definition; they gather an indefinite number of arguments into a real Array.

What is the syntax for Spread Syntax, and what are its common uses?;;...iterable (e.g., ...myArray); it expands an iterable into individual elements, useful for array/object copying, concatenation, and passing arguments to functions.

What is the main difference between Rest Parameters and Spread Syntax?;;Rest parameters gather elements into an array (in function definitions), while spread syntax expands an iterable into individual elements (in array/object literals or function calls).