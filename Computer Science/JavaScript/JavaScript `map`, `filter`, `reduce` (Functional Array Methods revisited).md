---
memory: to_finish
tags:
  - learning
language:
  - JavaScript
review-date: 2025-11-25
last-reviewed: 2025-10-17
scheda: done
visit-count: 1
confidence-level: 1
consecutive-correct: 0
last-struggle-date: 2025-10-17
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

<mark style="background: #D2B3FFA6;">Traditional for loops, while powerful, can be verbose and imperative.</mark> They require you to manually manage the loop's state (the counter, the exit condition) and explicitly state how to perform an operation. This can lead to code that is harder to read, more prone to off-by-one errors, and often involves mutating data, which can cause unintended side effects.

<mark style="background: #FFB86CA6;">map, filter, and reduce solve this by providing a declarative, functional approach to array manipulation. Instead of describing the step-by-step mechanics of a loop, you declare your intent: "transform this data," "select a subset of this data," or "aggregate this data into a single value."</mark> This leads to cleaner, more predictable, and more maintainable code, which are cornerstones of modern JavaScript development.

# **Core Explanation:**
---

<mark style="background: #BBFABBA6;">map, filter, and reduce are higher-order functions available on the Array.prototype in JavaScript.</mark> <mark style="background: #BBFABBA6;">They each iterate over an array and perform an operation</mark>, but for different purposes. A key characteristic is their ==**immutability**: they do not modify the original array; instead, they return a new array or value.==
## Array.prototype.map

- **What it is:** A method that creates a **new array** by calling a provided function on every element of the original array.

- **How it works:** <mark style="background: #FF5582A6;">It iterates through each element, applies the callback function to it, and collects the return value of each function call into a new array. The resulting array will always have the same length as the original array.</mark>

- **Primary Use:** Transforming data. For example, c<mark style="background: #FF5582A6;">onverting an array of user objects into an array of their email addresses.</mark>

## Array.prototype.filter

- **What it is:** A method that creates a **new array** containing only the elements from the <mark style="background: #BBFABBA6;">original array that pass a test</mark> (i.e., for which the callback function returns a truthy value).

- **How it works:** It iterates through each element and executes the callback function. If the callback returns true, t<mark style="background: #BBFABBA6;">he element is included in the new array; otherwise, it is skipped</mark>. The new array's length will be less than or equal to the original's.

- **Primary Use:** <mark style="background: #BBFABBA6;">Selecting a subset of data. For example, getting a list of all users who are currently active.</mark>

## Array.prototype.reduce

- **What it is:** A method that executes a "reducer" function on each element of the array, resulting in a **single output value**.

- **How it works:** It iterates through the array, applying a callback that takes two main arguments: <mark style="background: #ABF7F7A6;">an accumulator and the currentValue</mark>. <mark style="background: #ABF7F7A6;">The accumulator is the value returned from the previous iteration, and it "accumulates" the result. An optional initial value for the accumulator can be provided.</mark>

- **Primary Use:** <mark style="background: #ABF7F7A6;">Aggregating data. It is the most versatile of the three and can be used for summing numbers, flattening arrays, or grouping objects.</mark>

# **Related Concepts:**
---

- **Functional Programming (FP):** map, filter, and reduce are core concepts in functional programming. They align with FP principles like using pure functions (the callbacks should ideally be side-effect-free) and favoring immutability (not changing the original data).

- **Declarative vs. Imperative Programming:**

 - **Imperative** (e.g., for loops) is about describing how to do something.

 - **Declarative** (e.g., map, filter, reduce) is about describing what result you want. This often leads to more readable and abstract code.

- **Immutability:** This is the principle of not changing data after it's created. Since these methods return new arrays instead of modifying the original, they help prevent bugs caused by unexpected side effects, making code more predictable.

- **Method Chaining:** Because map and filter both return new arrays, they can be "chained" together to perform a sequence of operations in a single, elegant statement. For example, you can first filter an array and then map over the results.

# **Examples:**
---

```js
// Sample data for our examples
const users = [
 { id: 1, name: 'Alice', age: 30, isActive: true },
 { id: 2, name: 'Bob', age: 25, isActive: false },
 { id: 3, name: 'Charlie', age: 35, isActive: true },
 { id: 4, name: 'Diana', age: 28, isActive: true },
];

//
---
1. map Example
---
// Purpose: To get an array containing only the names of the users.
// The map method iterates over each 'user' object in the 'users' array.
// For each user, it executes the arrow function, which returns the user's name.
// The result is a new array of strings.
const userNames = users.map(user => user.name);

console.log('User Names (map):', userNames);
// Expected Output: ['Alice', 'Bob', 'Charlie', 'Diana']

//
---
2. filter Example
---
// Purpose: To get a new array with only the active users.
// The filter method iterates over each 'user' object.
// The callback function returns 'true' if the user's 'isActive' property is true,
// and 'false' otherwise. Only the users for whom the function returns true are included in the new array.
const activeUsers = users.filter(user => user.isActive);

console.log('Active Users (filter):', activeUsers);
// Expected Output: An array of objects for Alice, Charlie, and Diana.

//
---
3. reduce Example
---
// Purpose: To calculate the total age of all users.
// The reduce method iterates through the array to produce a single value.
// It takes a callback function and an initial value for the accumulator (in this case, 0).
// 'totalAge' is the accumulator, 'user' is the current element.
// In each step, we return the current totalAge plus the current user's age.
// This new value becomes the 'totalAge' for the next iteration.
const totalAge = users.reduce((totalAge, user) => totalAge + user.age, 0);

console.log('Total Age (reduce):', totalAge);
// Expected Output: 118 (30 + 25 + 35 + 28)

//
---
4. Chaining Example
---
// Purpose: To get the names of active users who are over 29 years old.
// This demonstrates the power of combining these methods.
const namesOfOlderActiveUsers = users
 // Step 1: Filter the array to get only active users.
 // The result of this filter is a new, smaller array.
 .filter(user => user.isActive)

 // Step 2: Filter the result of the first filter again.
 // The result is an even smaller array containing only users over 29.
 .filter(user => user.age > 29)

 // Step 3: Map over the final filtered array.
 // Transform the remaining user objects into an array of just their names.
 .map(user => user.name);

console.log('Chained Result:', namesOfOlderActiveUsers);
// Expected Output: ['Alice', 'Charlie']
```

# **Flashcards:**
---

What does the Array.prototype.map method do?;;It creates a new array by calling a provided function on every element in the original array. The new array always has the same length as the original.
What is the purpose of the Array.prototype.filter method?;;It creates a new array containing all elements that pass the test implemented by the provided callback function. The new array's length is less than or equal to the original's.
What does the Array.prototype.reduce method do?;;It executes a reducer function on each element of the array, resulting in a single, accumulated output value.
Do map, filter, and reduce modify the original array?;;No, they practice immutability. They always return a new array (or a single value for reduce), leaving the original array unchanged.
Why can map and filter be chained together?;;Because they both return a new array, the return value of one can be used as the array for the next method call, allowing for sequential transformations.
What are the two main arguments passed to the reduce callback function?;;The accumulator (the value from the previous iteration or the initial value) and the currentValue (the element currently being processed).