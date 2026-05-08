---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date: ""
last-reviewed: 2025-08-29
scheda: done
visit-count: 5
confidence-level: 2
consecutive-correct: 6

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
An array is a non-primitive data type that can hold a series of values. Non-primitive data types differ from primitive data types in that they can hold more complex data. Primitive data types like strings and numbers can only hold one value at a time.

Arrays are denoted using square brackets (``). You can access the values inside an array using the index of the value. An index is a number representing the position of the value in the array, starting from `0` for the first value.

You can access the value using bracket notation, such as `array`.

Arrays are special in that they are considered mutable. This means you can change the value at an index directly.

You can make use of the `.length` property of an array - this returns the number of elements in the array. To get the last element of any array, you can use the following syntax:

```js
array[array.length - 1]
```

`array.length` returns the number of elements in the array. By subtracting `1`, you get the index of the last element in the array. You can apply this same concept to your `rows` array.

# **Related Concepts:**

---
- **Primitive Data Types:** Basic data types that hold a single value directly, such as `string`, `number`, `boolean`, `null`, `undefined`, `symbol`, and `bigint`.
- **Non-Primitive Data Types (Reference Types):** Data types that can hold collections of data and are stored by reference, such as `objects`, `arrays`, and `functions`. When you assign a non-primitive type, you are assigning a reference to its memory location, not the value itself.
- **Zero-based Indexing:** The common convention in many programming languages (including JavaScript) where the first element of a sequence (like an array) is at index 0, the second at index 1, and so on.
- **Mutabilty:** The ability of an object to be changed after it is created. Arrays are mutable, meaning their contents can be modified (elements added, removed, or changed) without creating a new array.
- **Array Methods:** Built-in functions that can be called on array objects to perform common operations like adding/removing elements, iterating, transforming, and searching.

Examples include `push`, `pop`, `shift`, `unshift`, `splice`, `slice`, `forEach`, `map`, `filter`, `reduce`, etc.
- **Iteration:** The process of looping through the elements of an array to perform an operation on each element. This can be done using `for` loops (`for...of`, `for...in`), or array methods like `forEach`.

# **Examples:**

---
```Javascript
// 1. Array Declaration and Initialization
// Arrays are created using square brackets .
// They can be empty, or initialized with various data types.
const emptyArray = ; // An array with no elements.
const numbers = [1, 2, 3, 4, 5]; // An array containing only numbers.
const fruits = ["apple", "banana", "cherry"]; // An array containing only strings.
// Arrays can hold elements of different data types simultaneously.
const mixedData = [1, "hello", true, { key: "value" }, null]; // An array with a number, string, boolean, object, and null.
// Arrays can also contain other arrays, creating multi-dimensional or nested arrays.
const nestedArray = [[1, 2], [3, 4]]; // An array where each element is another array.

console.log("
---
1. Array Declaration and Initialization
---
");
console.log("Empty Array:", emptyArray);
console.log("Numbers Array:", numbers);
console.log("Fruits Array:", fruits);
console.log("Mixed Data Array:", mixedData);
console.log("Nested Array:", nestedArray);

// 2. Accessing Array Elements (Zero-based Indexing)
// Elements in an array are accessed using their index, which starts from 0 for the first element.
// This is known as zero-based indexing.
console.log("\n
---
2. Accessing Array Elements
---
");
console.log("First number:", numbers); // Accesses the element at index 0 (which is 1).
console.log("Third fruit:", fruits); // Accesses the element at index 2 (which is "cherry").
console.log("Value at index 3 in mixedData:", mixedData); // Accesses the object { key: "value" }.
// Accessing an index that does not exist in the array will return `undefined`.
console.log("Accessing an element out of bounds (index 10):", numbers); // Output: undefined

// 3. Array Mutability (Changing elements)
// Arrays are mutable, meaning their contents can be modified after they are created.
// You can change an element by assigning a new value to a specific index.
console.log("\n
---
3. Array Mutability
---
");
let colors = ["red", "green", "blue"];
console.log("Original colors:", colors);

colors = "yellow"; // The element at index 1 ("green") is replaced with "yellow".
console.log("Colors after changing index 1:", colors); // Output: ["red", "yellow", "blue"]

// You can also add new elements by assigning a value to an index beyond the current length.
// If there's a gap between the last existing element and the new index, those intermediate positions
// will become "empty slots" (often represented as `empty` or `undefined` depending on context).
colors = "purple"; // Adds "purple" at index 3.
console.log("Colors after adding at index 3:", colors); // Output: ["red", "yellow", "blue", "purple"] (no empty slot here as 3 was immediately after 2)
// Example with a gap:
colors = "orange";
console.log("Colors after adding at index 5:", colors); // Output: ["red", "yellow", "blue", "purple", empty, "orange"]

// 4. Using the .length Property
// The `.length` property returns the number of elements in an array.
// It's always one greater than the highest index in the array.
console.log("\n
---
4. Using the .length Property
---
");
console.log("Length of numbers array:", numbers.length); // Output: 5 (elements from index 0 to 4)
console.log("Length of fruits array:", fruits.length); // Output: 3
console.log("Length of colors array:", colors.length); // Output: 6 (due to the gap created by assigning at index 5)

// A common pattern to get the last element of an array:
// Since indexes are 0-based, the last element's index is `length - 1`.
console.log("Last element of fruits array:", fruits[fruits.length - 1]); // fruits[3 - 1] -> fruits -> "cherry"
console.log("Last element of colors array:", colors[colors.length - 1]); // colors[6 - 1] -> colors -> "orange"

// 5. Common Array Methods
// JavaScript provides a rich set of built-in methods for manipulating arrays.
console.log("\n
---
5. Common Array Methods
---
");
let planets = ["Mercury", "Venus", "Earth"];
console.log("Initial planets:", planets);

// push: Adds one or more elements to the *end* of an array and returns the new length of the array.
planets.push("Mars", "Jupiter");
console.log("After push (add to end):", planets); // planets is now ["Mercury", "Venus", "Earth", "Mars", "Jupiter"]

// pop: Removes the *last* element from an array and returns that removed element.
const lastPlanet = planets.pop;
console.log("After pop (remove from end):", planets); // planets is now ["Mercury", "Venus", "Earth", "Mars"]
console.log("Removed planet:", lastPlanet); // Output: "Jupiter"

// shift: Removes the *first* element from an array and returns that removed element.
const firstPlanet = planets.shift;
console.log("After shift (remove from start):", planets); // planets is now ["Venus", "Earth", "Mars"]
console.log("Removed planet:", firstPlanet); // Output: "Mercury"

// unshift: Adds one or more elements to the *beginning* of an array and returns the new length.
planets.unshift("Saturn", "Uranus");
console.log("After unshift (add to start):", planets); // planets is now ["Saturn", "Uranus", "Venus", "Earth", "Mars"]

// splice: A powerful method that can add, remove, or replace elements anywhere in an array.
// Syntax: `array.splice(startIndex, deleteCount, item1, item2, ...)`
// - `startIndex`: The index at which to start changing the array.
// - `deleteCount`: The number of elements to remove. If 0, no elements are removed.
// - `item1, item2, ...`: The elements to add to the array, starting at the startIndex.
let animals = ["cat", "dog", "bird", "fish"];
console.log("Initial animals:", animals);
animals.splice(1, 1, "rabbit", "hamster"); // Starting at index 1, remove 1 element ("dog"), then add "rabbit" and "hamster".
console.log("After splice (replace & add):", animals); // Output: ["cat", "rabbit", "hamster", "bird", "fish"]

// slice: Returns a *shallow copy* of a portion of an array into a *new* array object.
// The original array remains unchanged.
// Syntax: `array.slice(startIndex, endIndex (exclusive))`
// - `startIndex`: (Optional) The index at which to begin extraction. Defaults to 0.
// - `endIndex`: (Optional) The index before which to end extraction. The element at this index is NOT included. Defaults to array.length.
const slicedAnimals = animals.slice(1, 3); // Extracts elements from index 1 up to (but not including) index 3.
console.log("Original animals (unchanged by slice):", animals); // Output: ["cat", "rabbit", "hamster", "bird", "fish"]
console.log("Sliced animals (new array):", slicedAnimals); // Output: ["rabbit", "hamster"]

// 6. Iterating with forEach
// `forEach` executes a provided callback function once for each array element.
// It does not create a new array.
console.log("\n
---
6. Iterating with forEach
---
");
fruits.forEach((fruit, index) => { // The callback receives the current element and its index.
 console.log(`Fruit at index ${index}: ${fruit}`);
});
// Output:
// Fruit at index 0: apple
// Fruit at index 1: banana
// Fruit at index 2: cherry

// 7. Transforming with map
// `map` creates a *new array* populated with the results of calling a provided function
// on every element in the calling array. It does not modify the original array.
console.log("\n
---
7. Transforming with map
---
");
const doubledNumbers = numbers.map(num => num * 2); // For each 'num' in 'numbers', return 'num * 2'.
console.log("Original numbers:", numbers); // Output: [1, 2, 3, 4, 5] (unchanged)
console.log("Doubled numbers (new array):", doubledNumbers); // Output: [2, 4, 6, 8, 10]

// 8. Filtering with filter
// `filter` creates a *new array* with all elements that pass the test implemented
// by the provided function. Elements for which the callback function returns `true` are included.
console.log("\n
---
8. Filtering with filter
---
");
const evenNumbers = numbers.filter(num => num % 2 === 0); // Keep elements where 'num % 2 === 0' is true.
console.log("Original numbers:", numbers); // Output: [1, 2, 3, 4, 5] (unchanged)
console.log("Even numbers (new array):", evenNumbers); // Output: [2, 4]

// 9. Reducing with reduce
// `reduce` executes a reducer function (that you provide) on each element of the array,
// resulting in a single output value (e.g., sum, product, single object).
// Syntax: `array.reduce((accumulator, currentValue, currentIndex, array) => { ... }, initialValue)`
// - `accumulator`: The value resulting from the previous callback invocation, or `initialValue`.
// - `currentValue`: The current element being processed in the array.
console.log("\n
---
9. Reducing with reduce
---
");
const sumOfNumbers = numbers.reduce((sum, num) => sum + num, 0); // 'sum' starts at 0, then adds each 'num'.
console.log("Original numbers:", numbers); // Output: [1, 2, 3, 4, 5] (unchanged)
console.log("Sum of numbers:", sumOfNumbers); // Output: 15 (0+1+2+3+4+5)

// 10. Searching with indexOf and includes
console.log("\n
---
10. Searching with indexOf / includes
---
");
// indexOf: Returns the *first* index at which a given element can be found in the array.
// Returns -1 if the element is not present.
console.log("'banana' is at index:", fruits.indexOf("banana")); // Output: 1
console.log("'grape' is at index:", fruits.indexOf("grape")); // Output: -1

// includes: Determines whether an array includes a certain value among its entries,
// returning `true` or `false`.
console.log("'cherry' is included:", fruits.includes("cherry")); // Output: true
console.log("'kiwi' is included:", fruits.includes("kiwi")); // Output: false
```

# **Flashcards:**

---
How are arrays declared and what is their key characteristic regarding mutability?;; Arrays are declared using square brackets `` and are mutable, meaning their contents can be changed after creation.

How do you access the first and last elements of a JavaScript array?;; Access the first element using `array`. Access the last element using `array[array.length - 1]`.

Name three common array methods used for modifying an array in place.;;`push` (add to end), `pop` (remove from end), `shift` (remove from start), `unshift` (add to start), `splice` (add/remove/replace at any index). (Choose any three)