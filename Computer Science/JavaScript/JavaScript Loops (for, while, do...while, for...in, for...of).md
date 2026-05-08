---
memory: to_finish
tags:
  - learned
language:
  - JavaScript
review-date:
last-reviewed: 2025-09-04
scheda: done
visit-count: 4
confidence-level: 2.5
consecutive-correct: 3
last-struggle-date: 2025-07-25
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
JavaScript loops solve the fundamental problem of executing repetitive tasks without writing redundant code. <mark style="background: #D2B3FFA6;">They enable developers to iterate over data structures, process collections of items, perform calculations multiple times, and automate repetitive operations efficiently. Loops are essential in JavaScript because they provide the foundation for data processing, DOM manipulation, array operations, and algorithmic implementations</mark>. Without loops, developers would need to write hundreds or thousands of lines of duplicate code for tasks like processing user input, rendering lists, validating forms, or implementing search algorithms. They are crucial for creating dynamic, interactive web applications and handling the iterative nature of most programming problems.

# **Core Explanation:**
---
JavaScript loops are control structures that repeatedly execute a block of code until a specified condition is met. Each loop type serves different iteration scenarios and data structures.

**For Loop**: The most common loop, ideal when you know the exact number of iterations needed. It consists of three parts: initialization, condition, and increment/decrement expression.

**While Loop**: Executes code as long as a condition remains true. The condition is checked before each iteration, so the loop body might never execute if the condition is initially false.

**Do...While Loop**: Similar to while loop but guarantees at least one execution because the condition is checked after the loop body runs.

**For...In Loop**: <mark style="background: #FF5582A6;">Iterates over enumerable properties of objects, including inherited properties from the prototype chain. It's designed for object property enumeration.</mark>

To walk over all keys of an object, there exists a special form of the loop: `for..in`. This is a completely different thing from the `for()` construct that we studied before.

The syntax:

```javascript
for (key in object) {
  // executes the body for each key among object properties
}
```

For instance, let’s output all properties of `user`:

```javascript
let user = {
  name: "John",
  age: 30,
  isAdmin: true
};

for (let key in user) {
  // keys
  alert( key );  // name, age, isAdmin
  // values for the keys
  alert( user[key] ); // John, 30, true
}
```

Note that all “for” constructs allow us to declare the looping variable inside the loop, like `let key` here.

Also, we could use another variable name here instead of `key`. For instance, `"for (let prop in obj)"` is also widely used.

**For...Of Loop**: Introduced in ES6, it iterates over iterable objects (arrays, strings, maps, sets) and provides direct access to values rather than indices or keys.

Key characteristics include:
- **Control Flow**: Loops alter the normal sequential execution of code
- **Iteration Variable**: Most loops use a variable to track progress or access elements
- **Termination Condition**: All loops must have a way to stop executing to prevent infinite loops
- **Scope**: Variables declared within loops have block scope (with let/const)
- **Performance**: Different loops have varying performance characteristics depending on the use case

Loops work by evaluating a condition before or after executing a code block, then repeating this process until the condition becomes false or a break statement is encountered.

# **Related Concepts:**
---
**Array Methods (forEach, map, filter, reduce)**: Higher-order functions that provide functional programming alternatives to traditional loops, often more readable and less error-prone for array operations.

**Break and Continue Statements**: Control statements that modify loop behavior - break exits the loop entirely, while continue skips the current iteration and moves to the next one.

**Iterators and Iterables**: ES6 concepts that define how objects can be iterated over, providing the foundation for for...of loops and enabling custom iteration behavior.

**Recursion**: An alternative to loops where a function calls itself repeatedly, often used for tree traversal and divide-and-conquer algorithms.

**Asynchronous Iteration**: Modern JavaScript concepts like async/await in loops and for await...of for handling asynchronous operations within iterations.

**Nested Loops**: Loops within loops, commonly used for multi-dimensional data processing but requiring careful consideration of time complexity.

**Loop Optimization**: Techniques for improving loop performance, including loop unrolling, caching array lengths, and avoiding unnecessary operations within loop bodies.

**Scope and Closures**: How variable scope works within loops, particularly important when creating functions inside loops or using setTimeout/setInterval.

# **Examples:**
---
```javascript
// DO...WHILE LOOP EXAMPLES
// Executes at least once, then checks condition

function demonstrateDoWhileLoop() {
    // Menu system - always show menu at least once
    let choice;
    
    do {
        // This block executes at least once before checking condition
        console.log("Menu Options:");
        console.log("1. Option A");
        console.log("2. Option B");
        console.log("3. Exit");
        
        // Simulate user choice (in real app, this would be actual input)
        choice = Math.floor(Math.random() * 4) + 1; // Random 1-4
        console.log(`User selected: ${choice}`);
        
        // Process choice
        switch(choice) {
            case 1:
                console.log("Executing Option A");
                break;
            case 2:
                console.log("Executing Option B");
                break;
            case 3:
                console.log("Exiting...");
                break;
            default:
                console.log("Invalid choice, try again");
        }
    } while (choice !== 3); // Continue until user chooses exit
    
    // Password validation example
    let password;
    let isValid = false;
    
    do {
        // Always prompt for password at least once
        password = "user123"; // Simulate password input
        
        // Validate password (at least 6 characters, contains number)
        if (password.length >= 6 && /\d/.test(password)) {
            isValid = true;
            console.log("Password accepted!");
        } else {
            console.log("Password must be at least 6 characters and contain a number");
            // In real app, you'd prompt for new password here
            break; // Exit for demo purposes
        }
    } while (!isValid); // Continue until valid password entered
}

// FOR...IN LOOP EXAMPLES
// Iterates over object properties and array indices

function demonstrateForInLoop() {
    // Object property iteration
    const person = {
        name: "John Doe",
        age: 30,
        city: "New York",
        occupation: "Developer"
    };
    
    console.log("Person properties:");
    // for...in iterates over property names (keys)
    for (let property in person) {
        // property is the key name as a string
        // person[property] accesses the value using bracket notation
        console.log(`${property}: ${person[property]}`);
    }
    
    // Array iteration with for...in (shows indices, not recommended)
    const fruits = ["apple", "banana", "orange"];
    
    console.log("Array indices with for...in:");
    for (let index in fruits) {
        // index is a string, not a number!
        console.log(`Index ${index} (type: ${typeof index}): ${fruits[index]}`);
    }
    
    // Object with inherited properties
    function Animal(name) {
        this.name = name;
    }
    Animal.prototype.species = "Unknown";
    
    const dog = new Animal("Buddy");
    dog.breed = "Golden Retriever";
    
    console.log("All enumerable properties (including inherited):");
    for (let prop in dog) {
        console.log(`${prop}: ${dog[prop]}`);
        // This will show: name, breed, and species (from prototype)
    }
    
    // Filter out inherited properties
    console.log("Own properties only:");
    for (let prop in dog) {
        // hasOwnProperty checks if property belongs to object itself
        if (dog.hasOwnProperty(prop)) {
            console.log(`${prop}: ${dog[prop]}`);
            // This will show only: name and breed
        }
    }
}

// FOR...OF LOOP EXAMPLES
// Iterates over iterable objects (arrays, strings, maps, sets)

function demonstrateForOfLoop() {
    // Array iteration - cleaner than for...in for arrays
    const colors = ["red", "green", "blue", "yellow"];
    
    console.log("Colors using for...of:");
    // for...of gives direct access to values, not indices
    for (let color of colors) {
        console.log(color); // Direct access to array elements
    }
    
    // String iteration - treats string as array of characters
    const message = "Hello";
    
    console.log("Characters in message:");
    for (const char of message) // you can use const because the variable only exists for a single iteration, not during the entire loop. 
    {
        console.log(char); // Each character: H, e, l, l, o
    }
    
    // Array with index access using entries()
    console.log("Colors with indices:");
    for (let [index, color] of colors.entries()) {
        // entries() returns [index, value] pairs
        // Destructuring assignment extracts index and color
        console.log(`${index}: ${color}`);
    }
    
    // Set iteration - for...of works with ES6 collections
    const uniqueNumbers = new Set([1, 2, 3, 2, 1, 4]);
    
    console.log("Unique numbers from Set:");
    for (let number of uniqueNumbers) {
        console.log(number); // Outputs: 1, 2, 3, 4 (duplicates removed)
    }
    
    // Map iteration - key-value pairs
    const userRoles = new Map([
        ["john", "admin"],
        ["jane", "user"],
        ["bob", "moderator"]
    ]);
    
    console.log("User roles:");
    for (let [username, role] of userRoles) {
        // Map entries are [key, value] pairs
        console.log(`${username}: ${role}`);
    }
    
    // NodeList iteration (common in DOM manipulation)
    // This would work in browser environment:
    /*
    const divElements = document.querySelectorAll('div');
    for (let div of divElements) {
        // Direct access to each DOM element
        console.log(div.textContent);
    }
    */
}

// PRACTICAL EXAMPLE: Data Processing with Different Loop Types
function processUserData() {
    const users = [
        { id: 1, name: "Alice", active: true, loginCount: 5 },
        { id: 2, name: "Bob", active: false, loginCount: 2 },
        { id: 3, name: "Charlie", active: true, loginCount: 8 },
        { id: 4, name: "Diana", active: true, loginCount: 3 }
    ];
    
    // for...of loop for clean array iteration
    console.log("Active users:");
    for (let user of users) {
        if (user.active) {
            console.log(`${user.name} (${user.loginCount} logins)`);
        }
    }
    
    // Traditional for loop when you need index
    console.log("User processing with index:");
    for (let i = 0; i < users.length; i++) {
        // Use index for position-dependent operations
        console.log(`Processing user ${i + 1}/${users.length}: ${users[i].name}`);
    }
    
    // for...in loop for object property analysis
    console.log("User object structure:");
    if (users.length > 0) {
        for (let property in users[0]) {
            console.log(`Property: ${property}, Type: ${typeof users[0][property]}`);
        }
    }
    
    // while loop for conditional processing
    let currentIndex = 0;
    let activeUsersFound = 0;
    const maxActiveUsers = 2;
    
    console.log(`Finding first ${maxActiveUsers} active users:`);
    while (currentIndex < users.length && activeUsersFound < maxActiveUsers) {
        if (users[currentIndex].active) {
            console.log(`Found active user: ${users[currentIndex].name}`);
            activeUsersFound++;
        }
        currentIndex++;
    }
}

// Run all demonstrations
demonstrateForLoop();
demonstrateWhileLoop();
demonstrateDoWhileLoop();
demonstrateForInLoop();
demonstrateForOfLoop();
processUserData();
````

```javascript
// ADVANCED LOOP PATTERNS AND BEST PRACTICES
// Common patterns, performance considerations, and potential pitfalls

// Nested loops for 2D array processing
function demonstrateNestedLoops() {
    // Creating a multiplication table
    const size = 5;
    const multiplicationTable = [];
    
    // Outer loop for rows
    for (let row = 1; row <= size; row++) {
        const currentRow = [];
        
        // Inner loop for columns
        for (let col = 1; col <= size; col++) {
            // Each iteration of inner loop runs once per outer loop iteration
            currentRow.push(row * col);
        }
        
        multiplicationTable.push(currentRow);
    }
    
    // Display the table
    console.log("Multiplication Table:");
    for (let row of multiplicationTable) {
        console.log(row.join("\t")); // Tab-separated values
    }
}

// Loop control with break and continue
function demonstrateLoopControl() {
    const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    
    console.log("Finding first number greater than 5:");
    for (let number of numbers) {
        if (number <= 5) {
            continue; // Skip current iteration, move to next
        }
        
        console.log(`Found: ${number}`);
        break; // Exit loop entirely
    }
    
    // Labeled break for nested loops
    outerLoop: for (let i = 1; i <= 3; i++) {
        console.log(`Outer loop iteration: ${i}`);
        
        for (let j = 1; j <= 3; j++) {
            if (i === 2 && j === 2) {
                console.log("Breaking out of both loops");
                break outerLoop; // Break out of labeled outer loop
            }
            console.log(`  Inner loop iteration: ${j}`);
        }
    }
}

// Performance considerations and optimizations
function demonstrateLoopOptimization() {
    const largeArray = new Array(1000000).fill(0).map((_, i) => i);
    
    // POOR: Recalculating array length every iteration
    console.time("Poor loop performance");
    for (let i = 0; i < largeArray.length; i++) {
        // largeArray.length is evaluated every iteration
        largeArray[i] = largeArray[i] * 2;
    }
    console.timeEnd("Poor loop performance");
    
    // BETTER: Cache array length
    console.time("Optimized loop performance");
    const length = largeArray.length; // Cache length outside loop
    for (let i = 0; i < length; i++) {
        largeArray[i] = largeArray[i] * 2;
    }
    console.timeEnd("Optimized loop performance");
    
    // BEST: Use appropriate array methods when possible
    console.time("Array method performance");
    const doubled = largeArray.map(num => num * 2);
    console.timeEnd("Array method performance");
}

// Common pitfalls and how to avoid them
function demonstrateCommonPitfalls() {
    // PITFALL 1: Infinite loops
    console.log("Demonstrating controlled 'infinite' loop:");
    let counter = 0;
    while (true) {
        counter++;
        console.log(`Iteration ${counter}`);
        
        // Always have an exit condition!
        if (counter >= 3) {
            break;
        }
    }
    
    // PITFALL 2: Modifying array while iterating
    console.log("Safe array modification during iteration:");
    let items = [1, 2, 3, 4, 5];
    
    // WRONG: Modifying array while iterating forward can skip elements
    // for (let i = 0; i < items.length; i++) {
    //     if (items[i] % 2 === 0) {
    //         items.splice(i, 1); // This shifts remaining elements
    //     }
    // }
    
    // RIGHT: Iterate backwards when removing elements
    for (let i = items.length - 1; i >= 0; i--) {
        if (items[i] % 2 === 0) {
            items.splice(i, 1); // Safe because we're going backwards
        }
    }
    console.log("Odd numbers remaining:", items);
    
    // PITFALL 3: Closure in loops (classic interview question)
    console.log("Closure in loops - the classic problem:");
    
    // PROBLEM: All functions will log 3 (the final value of i)
    const functions = [];
    for (var i = 0; i < 3; i++) {
        functions.push(function() {
            console.log("var i:", i); // i is 3 for all functions
        });
    }
    
    // SOLUTION 1: Use let instead of var (block scope)
    const functionsFixed1 = [];
    for (let i = 0; i < 3; i++) {
        functionsFixed1.push(function() {
            console.log("let i:", i); // Each function captures its own i
        });
    }
    
    // SOLUTION 2: Use IIFE (Immediately Invoked Function Expression)
    const functionsFixed2 = [];
    for (var i = 0; i < 3; i++) {
        functionsFixed2.push((function(index) {
            return function() {
                console.log("IIFE index:", index);
            };
        })(i)); // Pass current value of i to IIFE
    }
    
    // Execute all functions to see the difference
    console.log("Executing functions created with var:");
    functions.forEach(fn => fn());
    
    console.log("Executing functions created with let:");
    functionsFixed1.forEach(fn => fn());
    
    console.log("Executing functions created with IIFE:");
    functionsFixed2.forEach(fn => fn());
}

// Run advanced demonstrations
demonstrateNestedLoops();
demonstrateLoopControl();
demonstrateLoopOptimization();
demonstrateCommonPitfalls();
```

# **Flashcards:**
---
What are the key differences between for...in and for...of loops in JavaScript?;; for...in iterates over enumerable property names (keys) of objects and returns indices as strings for arrays, including inherited properties. for...of iterates over iterable objects (arrays, strings, maps, sets) and returns values directly, only working with objects that implement the iterable protocol.

When should you use a do...while loop instead of a while loop?;; Use do...while when you need the loop body to execute at least once before checking the condition, such as menu systems, user input validation, or any scenario where you want to perform an action first and then decide whether to repeat it based on the result.

What is the classic closure-in-loops problem and how do you solve it?;; When using var in a for loop and creating functions inside the loop, all functions reference the same variable and will use its final value. Solutions include: (1) using let instead of var for block scope, (2) using an IIFE to capture the current value, or (3) using bind() to create a new function with the current value bound to it.