---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date: ""
last-reviewed: 2025-08-17
scheda: done
visit-count: 4
confidence-level: 2
consecutive-correct: 3
last-struggle-date: 2025-07-06

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
JavaScript array methods are essential tools for ==manipulating collections of data efficiently==. They solve the fundamental problem of adding, removing, and extracting elements from arrays without manually iterating through indices. These methods are crucial because arrays are one of the most commonly used data structures in JavaScript, and these methods provide clean, readable ways to perform common operations like managing dynamic lists, implementing queues and stacks, and processing data collections in web applications and algorithms.

# **Core Explanation:**

---
#

# *JavaScript provides several built-in methods for array manipulation:*

**<mark style="background:

# FF5582A6;">Mutating</mark> Methods (modify original array):**
>- **push**: Adds one or more elements to the end of an array and <mark style="background:

# BBFABBA6;">returns the new length</mark>
>- **pop**: Removes and <mark style="background:

# D2B3FFA6;">returns the last element</mark> from an array
>- **shift**: Removes and <mark style="background:

# D2B3FFA6;">returns the first element</mark> from an array
>- **unshift**: Adds one or more elements to the beginning of an array and <mark style="background:

# BBFABBA6;">returns the new length</mark>
>- **splice**: Changes array contents by removing/replacing existing elements and/or adding new elements at any position

**<mark style="background:

# FF5582A6;">Non-mutating</mark> Methods (return new array/values):**
- **slice**: <mark style="background:

# BBFABBA6;">Returns a shallow copy of a portion of an array into a new array</mark>, selected from start to end (end not included)

*Key characteristics: Push/pop work with the end of arrays (LIFO - Last In, First Out), shift/unshift work with the beginning (FIFO - First In, First Out), slice creates copies without modification, and splice is the most versatile for complex modifications.*

# **Related Concepts:**

---
#

# *These array methods connect to several important programming concepts:*

- **Stack Data Structure**: Push and pop implement stack behavior (LIFO)
- **Queue Data Structure**: Unshift and pop (or push and shift) implement queue behavior (FIFO)
- **Immutability**: Slice promotes immutable programming patterns by creating copies
- **Array Indexing**: Understanding how these methods affect array indices and length
- **Reference vs Value**: Mutating methods modify the original array reference, while slice creates new arrays
- **Functional Programming**: Methods like slice align with functional programming principles of avoiding side effects

# **Examples:**

---
```javascript
// Initialize sample array for demonstrations
let fruits = ['apple', 'banana', 'orange'];
let numbers = [1, 2, 3, 4, 5];

// PUSH - adds elements to the end of array
let originalLength = fruits.length; // 3
let newLength = fruits.push('grape', 'kiwi'); // Add multiple elements
console.log(fruits); // ['apple', 'banana', 'orange', 'grape', 'kiwi']
console.log(newLength); // 5 - returns new array length

// POP - removes and returns the last element
let removedFruit = fruits.pop; // Removes 'kiwi'
console.log(removedFruit); // 'kiwi' - returns the removed element
console.log(fruits); // ['apple', 'banana', 'orange', 'grape']

// SHIFT - removes and returns the first element
let firstFruit = fruits.shift; // Removes 'apple'
console.log(firstFruit); // 'apple' - returns the removed element
console.log(fruits); // ['banana', 'orange', 'grape'] - indices shift down

// UNSHIFT - adds elements to the beginning of array
let newLengthUnshift = fruits.unshift('mango', 'pear'); // Add to beginning
console.log(fruits); // ['mango', 'pear', 'banana', 'orange', 'grape']
console.log(newLengthUnshift); // 5 - returns new array length

// SLICE - creates a new array from a portion (non-mutating)
let slicedNumbers = numbers.slice(1, 4); // From index 1 to 3 (4 excluded)
console.log(slicedNumbers); // [2, 3, 4] - new array created
console.log(numbers); // [1, 2, 3, 4, 5] - original unchanged
let lastTwo = numbers.slice(-2); // Negative indices count from end
console.log(lastTwo); // [4, 5] - last two elements

// SPLICE - remove/replace/add elements at any position (mutating)
let colors = ['red', 'green', 'blue', 'yellow', 'purple'];

// Remove 2 elements starting at index 1
let removedColors = colors.splice(1, 2); // Remove 'green' and 'blue'
console.log(removedColors); // ['green', 'blue'] - returns removed elements
console.log(colors); // ['red', 'yellow', 'purple'] - original modified

// Insert elements without removing (deleteCount = 0)
colors.splice(2, 0, 'orange', 'pink'); // Insert at index 2, remove 0
console.log(colors); // ['red', 'yellow', 'orange', 'pink', 'purple']

// Replace elements (remove and add in one operation)
colors.splice(1, 2, 'cyan', 'magenta'); // Remove 2, add 2 at index 1
console.log(colors); // ['red', 'cyan', 'magenta', 'pink', 'purple']

// Practical example: Building a simple task manager
let tasks = ;

// Add new tasks (push)
tasks.push('Buy groceries', 'Walk the dog');
console.log(tasks); // ['Buy groceries', 'Walk the dog']

// Complete last task (pop)
let completedTask = tasks.pop;
console.log(`Completed: ${completedTask}`); // 'Completed: Walk the dog'

// Add urgent task to front (unshift)
tasks.unshift('Pay bills'); // Urgent task goes first
console.log(tasks); // ['Pay bills', 'Buy groceries']

// Get subset of tasks without modifying original (slice)
let tasksCopy = tasks.slice; // Copy entire array
console.log(tasksCopy); // ['Pay bills', 'Buy groceries']
````

# **Flashcards:**

---
What does the push method do and what does it return?;; Adds one or more elements to the end of an array and returns the new length of the array

What's the difference between pop and shift methods?;; pop removes and returns the last element from an array, while shift removes and returns the first element from an array

Which array methods modify the original array (mutating)?;; push, pop, shift, unshift, and splice all modify the original array

What does slice(1, 4) return from array [0, 1, 2, 3, 4, 5]?;; Returns [1, 2, 3] - elements from index 1 up to (but not including) index 4

How do you add elements to the beginning of an array?;; Use unshift method - it adds one or more elements to the beginning and returns the new array length

What's the syntax for splice and what does it do?;; array.splice(startIndex, deleteCount, item1, item2, ...) - removes/replaces existing elements and/or adds new elements at any position, returns array of removed elements