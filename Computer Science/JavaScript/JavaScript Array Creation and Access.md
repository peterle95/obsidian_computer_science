---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date: ""
last-reviewed: 2025-07-25
scheda: done
visit-count: 1
confidence-level: 1
consecutive-correct: 1

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
Arrays are fundamental data structures in almost every programming language, and JavaScript is no exception. They solve the problem of **storing and managing collections of related data in an ordered manner**. Instead of declaring numerous individual variables (e.g., `item1`, `item2`, `item3`), arrays allow you to store multiple values under a single variable name, making it efficient to operate on these collections. This is crucial for tasks like managing lists of users, product inventories, sequences of events, or any scenario where you need to group similar items. In computer science, arrays are a cornerstone for implementing more complex data structures and algorithms.

# **Core Explanation:**

---
An **array** in JavaScript is a single variable that stores an ordered collection of values. These values can be of any data type (numbers, strings, booleans, objects, even other arrays), and an array can contain a mix of different types. Arrays are **zero-indexed**, meaning the first element is at index 0, the second at index 1, and so on.

There are several ways to **create** an array:

1. **Array Literal (Preferred Method)**: This is the simplest and most common way to create an array. You enclose a comma-separated list of values within square brackets ``.

 - Example: `const myArray = [1, 'hello', true];`
2. **`new Array` Constructor**: You can also create an array using the `new Array` constructor.

 - With arguments:
 - If a single number argument is provided, it creates an empty array with that specified length (e.g., `new Array(5)` creates an array of 5 empty slots).
 - If multiple arguments are provided, they become the initial elements of the array (e.g., `new Array(1, 2, 3)` creates an array `[1, 2, 3]`).
 - Without arguments: `new Array` creates an empty array, similar to ``.

**Accessing** elements in an array is done using their **index** (their position in the array). You use square bracket notation `` with the index number after the array variable.

- Example: `myArray` would access the first element.

Arrays are dynamic, meaning their size can change after creation. You can add or remove elements, and their length will adjust accordingly.

# **Related Concepts:**

---
- **Objects**: While both arrays and objects store collections of data, their primary use cases differ. Arrays store ordered collections accessible by numerical indices, ideal for lists where order matters. Objects store unordered collections of key-value pairs, ideal for representing entities with named properties. You might use an array of objects (e.g., `[{name: 'Alice', age: 30}, {name: 'Bob', age: 25}]`).

- **`length` Property**: Every JavaScript array has a `length` property that returns the number of elements in the array. This is crucial for iterating over arrays and understanding their current size.

- **Array Methods**: JavaScript provides a rich set of built-in array methods (e.g., `push`, `pop`, `shift`, `unshift`, `splice`, `map`, `filter`, `forEach`, `reduce`). These methods allow you to efficiently manipulate, transform, and iterate over array elements, building upon the basic creation and access principles.

- **Iteration (Loops)**: Loops (like `for`, `for...of`, `forEach`) are used to process each element in an array. Understanding how to access individual elements by their index or value is fundamental to iterating through an array.

# **Examples:**

---
```js
//
---
Array Creation Examples
---
// 1. Using Array Literal (most common and recommended)
const fruits = ["Apple", "Banana", "Cherry"]; // Creates an array with three string elements
console.log(fruits); // Output: ["Apple", "Banana", "Cherry"]

const numbers = [10, 20, 30, 40, 50]; // Creates an array of numbers
console.log(numbers); // Output: [10, 20, 30, 40, 50]

const mixedArray = [1, "hello", true, { key: "value" }, [1, 2]]; // Arrays can hold mixed data types
console.log(mixedArray); // Output: [1, "hello", true, { key: "value" }, [1, 2]]

const emptyArrayLiteral = ; // Creates an empty array
console.log(emptyArrayLiteral); // Output:

// 2. Using new Array Constructor
const colors = new Array("Red", "Green", "Blue"); // Creates an array with initial elements
console.log(colors); // Output: ["Red", "Green", "Blue"]

const numbersWithLength = new Array(5); // Creates an array with 5 empty slots (not 5 as an element)
console.log(numbersWithLength); // Output: [empty × 5] (or similar representation of empty slots)
numbersWithLength = 100; // You can assign values to these slots
console.log(numbersWithLength); // Output: [100, empty × 4]

const emptyArrayConstructor = new Array; // Creates an empty array
console.log(emptyArrayConstructor); // Output:

//
---

Array Access Examples
---
// Accessing elements by index
console.log(fruits); // Accesses the first element (index 0) - Output: Apple
console.log(fruits); // Accesses the second element (index 1) - Output: Banana
console.log(fruits); // Accesses the third element (index 2) - Output: Cherry

// Accessing an index that does not exist returns undefined
console.log(fruits); // Output: undefined

// Modifying elements using index
fruits = "Orange"; // Changes the second element from "Banana" to "Orange"
console.log(fruits); // Output: ["Apple", "Orange", "Cherry"]

// Getting the length of an array
console.log(fruits.length); // Output: 3

// Accessing the last element using length
console.log(fruits[fruits.length - 1]); // Output: Cherry (Accesses the last element dynamically)

// Iterating over an array using a for loop
console.log("\nIterating through fruits:");
for (let i = 0; i < fruits.length; i++) {
 console.log(`Fruit at index ${i}: ${fruits[i]}`);
}
// Output:
// Fruit at index 0: Apple
// Fruit at index 1: Orange
// Fruit at index 2: Cherry
```

# **Flashcards:**

---
How do you create an array using the most common and recommended method in JavaScript?;; By using array literal notation: .

What is the index of the first element in a JavaScript array?;; The first element is at index 0.

How do you access an element in an array?;; By using bracket notation with the element's index, e.g., myArray[index].