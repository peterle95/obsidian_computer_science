---
memory: done
tags:
  - mastered
language:
  - JavaScript
review-date: ""
last-reviewed: 2025-07-07
scheda: done
visit-count: 2
confidence-level: 2
consecutive-correct: 2
---
--> check difference with c++

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
The fundamental problem JavaScript Control Flow solves is determining the order in which individual statements or instructions are executed in a program. By default, code runs sequentially, line by line. However, real-world applications often need to make decisions, repeat actions, or handle errors based on various conditions. Control flow mechanisms allow developers to dictate this non-sequential execution path.

It is critically important in computer science and JavaScript for several reasons:

- **Decision Making:** Enables programs to react dynamically to different inputs or states (e.g., "if a user is logged in, show their profile; otherwise, show a login form").
- **Repetition/Iteration:** Allows for efficient execution of repetitive tasks without writing redundant code (e.g., processing all items in a list, performing calculations a fixed number of times).
- **Error Handling:** Provides mechanisms to gracefully manage unexpected situations or errors, preventing program crashes and improving robustness.
- **Modularity and Organization:** Structures code into logical blocks (like functions) that can be executed on demand, improving readability and maintainability.
- **Algorithm Implementation:** Forms the backbone of virtually all algorithms, dictating the steps and conditions for solving computational problems.

Without control flow, programs would be linear and inflexible, severely limiting their functionality and ability to interact with dynamic environments.

# **Core Explanation:**

---
**Control Flow** refers to the order in which a computer executes statements in a script. In JavaScript, as in most programming languages, statements are typically executed from top to bottom, one after another. However, control flow statements allow you to alter this default sequential execution, enabling your program to make decisions, repeat actions, or jump to different parts of the code.

The main categories of control flow in JavaScript are:

1. **Conditional Statements:** These execute different blocks of code based on whether a specified condition evaluates to `true` or `false`.

 - **`if...else if...else`:** Executes a block of code if a condition is true, optionally executing another block if the first is false, and subsequent blocks for other conditions.
 - **`switch`:** Evaluates an expression and executes one of several code blocks (cases) that match the expression's value. It provides a more structured way to handle multiple `if...else if` conditions for a single variable.
2. **Looping Statements (Iteration):** These repeatedly execute a block of code as long as a specified condition is true or for a certain number of times.

 - **`for` loop:** Executes a block of code a specific number of times. It includes an initialization, a condition, and an update expression.
 - **`while` loop:** Executes a block of code as long as a specified condition is true. The condition is checked _before_ each iteration.
 - **`do...while` loop:** Similar to `while`, but the block of code is executed _at least once_ before the condition is checked.
 - **`for...in` loop:** Iterates over the enumerable properties of an object.
 - **`for...of` loop:** Iterates over iterable objects (like Arrays, Strings, Maps, Sets, etc.), typically accessing the values of the elements.
3. **Jump Statements:** These transfer control to another part of the program.

 - **`break`:** Terminates the current loop or `switch` statement immediately.
 - **`continue`:** Skips the rest of the current iteration of a loop and proceeds to the next iteration.
 - **`return`:** Exits the current function and optionally returns a value.
4. **Error Handling Statements:** These allow you to handle runtime errors gracefully.

 - **`try...catch...finally`:** `try` defines a block of code to be tested for errors. `catch` defines a block of code to be executed if an error occurs in the `try` block. `finally` defines a block of code that is executed regardless of whether an error occurred or not.
 - **`throw`:** Throws (generates) a user-defined exception.

# **Related Concepts:**

---
- **Boolean Logic:** Conditional statements rely heavily on boolean expressions (expressions that evaluate to `true` or `false`). Understanding logical operators (`&&`, `||`, `!`) is crucial for building complex conditions.
- **Functions:** Functions are essential for organizing code into reusable blocks. The `return` statement is a control flow mechanism specific to functions, allowing them to exit and potentially return a value. Function calls themselves represent a transfer of control.
- **Scope:** Control flow statements often define or affect the scope of variables. For example, variables declared with `let` or `const` inside a loop or `if` block are block-scoped, meaning they are only accessible within that block.
- **Asynchronous JavaScript (Callbacks, Promises, Async/Await):** While traditional control flow is synchronous, modern JavaScript heavily utilizes asynchronous patterns. These patterns introduce a different kind of "control flow" over time, where operations might not complete immediately. `async/await` provides a syntactic sugar that makes asynchronous code _look_ more like synchronous control flow.
- **Recursion:** An alternative to looping for repetitive tasks, where a function calls itself until a base case is met. It's another way to manage iterative processes, often used when dealing with tree-like data structures.
- **Exception Handling:** The `try...catch...finally` block is a specific type of control flow designed to manage errors and prevent program crashes, providing a structured way to react to unexpected events.

# **Examples:**

---
```js
//
---
1. Conditional Statements
---
// Example: if...else if...else
let temperature = 25;

if (temperature > 30) {
 console.log("It's scorching hot!"); // This block executes if temperature is > 30
} else if (temperature > 20) {
 console.log("It's warm and pleasant."); // This block executes if temperature is > 20 (and not > 30)
} else {
 console.log("It's a bit chilly."); // This block executes if none of the above conditions are met
}

// Example: switch statement
let dayOfWeek = "Wednesday";
let message;

switch (dayOfWeek) {
 case "Monday":
 message = "Start of the work week.";
 break; // 'break' is crucial to exit the switch after a match, preventing "fall-through"
 case "Friday":
 message = "Almost the weekend!";
 break;
 case "Saturday":
 case "Sunday": // Multiple cases can share the same code block
 message = "It's the weekend!";
 break;
 default:
 message = "Just another weekday."; // Default case if no other case matches
}
console.log(message); // Output: Just another weekday.

//
---
2. Looping Statements
---
// Example: for loop (iterate a fixed number of times)
console.log("\nFor Loop:");
for (let i = 0; i < 5; i++) {
 // i starts at 0, increments by 1 each time, and loop continues as long as i < 5
 console.log(`Count: ${i}`); // Outputs 0, 1, 2, 3, 4
}

// Example: while loop (iterate as long as a condition is true)
console.log("\nWhile Loop:");
let count = 0;
while (count < 3) {
 // Loop continues as long as 'count' is less than 3
 console.log(`Current count: ${count}`);
 count++; // Increment 'count' to eventually make the condition false
}
// Outputs: Current count: 0, Current count: 1, Current count: 2

// Example: do...while loop (guaranteed to run at least once)
console.log("\nDo...While Loop:");
let x = 0;
do {
 // This block executes once, then the condition is checked
 console.log(`Do-While count: ${x}`);
 x++;
} while (x < -1); // Condition is false, but the block ran once
// Outputs: Do-While count: 0

// Example: for...of loop (iterating over iterable values like arrays)
console.log("\nFor...Of Loop:");
const fruits = ["apple", "banana", "cherry"];
for (const fruit of fruits) {
 console.log(`Fruit: ${fruit}`);
}
// Outputs: Fruit: apple, Fruit: banana, Fruit: cherry

// Example: for...in loop (iterating over object properties)
console.log("\nFor...In Loop:");
const person = {
 name: "Alice",
 age: 30,
 city: "New York"
};
for (const key in person) {
 // Iterates over keys (property names) of the 'person' object
 console.log(`${key}: ${person[key]}`);
}
// Outputs: name: Alice, age: 30, city: New York

//
---
3. Jump Statements
---
// Example: break (exiting a loop early)
console.log("\nBreak Example:");
for (let i = 1; i <= 10; i++) {
 if (i === 5) {
 console.log("Breaking loop at 5.");
 break; // Terminates the loop immediately
 }
 console.log(i);
}
// Outputs: 1, 2, 3, 4, Breaking loop at 5.

// Example: continue (skipping current iteration)
console.log("\nContinue Example:");
for (let i = 1; i <= 5; i++) {
 if (i % 2 === 0) {
 // If 'i' is even, skip the rest of this iteration and go to the next
 continue;
 }
 console.log(`Odd number: ${i}`);
}
// Outputs: Odd number: 1, Odd number: 3, Odd number: 5

// Example: return (exiting a function)
function add(a, b) {
 if (typeof a !== 'number' || typeof b !== 'number') {
 return "Please provide numbers."; // Exits the function and returns this string
 }
 return a + b; // Exits the function and returns the sum
}
console.log(add(5, 3)); // Output: 8
console.log(add(5, "hello")); // Output: Please provide numbers.

//
---
4. Error Handling Statements
---
// Example: try...catch...finally
console.log("\nTry...Catch...Finally Example:");
function divide(numerator, denominator) {
 try {
 // Code that might throw an error
 if (denominator === 0) {
 throw new Error("Cannot divide by zero!"); // Throws a custom error
 }
 const result = numerator / denominator;
 console.log(`Result of division: ${result}`);
 } catch (error) {
 // Code to handle the error if one occurred in the 'try' block
 console.error(`An error occurred: ${error.message}`);
 } finally {
 // Code that will always execute, regardless of whether an error occurred
 console.log("Division attempt finished.");
 }
}

divide(10, 2); // Output: Result of division: 5, Division attempt finished.
divide(10, 0); // Output: An error occurred: Cannot divide by zero!, Division attempt finished.
```

# **Flashcards:**

---
What is JavaScript Control Flow?;; JavaScript Control Flow dictates the order in which statements in a program are executed, allowing for decision-making, repetition, and error handling.

Name the three main categories of control flow statements.;; The three main categories are Conditional Statements (e.g., `if`, `switch`), Looping Statements (e.g., `for`, `while`), and Jump Statements (e.g., `break`, `continue`, `return`).

What is the difference between `break` and `continue`?;; `break` immediately terminates the entire loop (or `switch` statement), while `continue` skips only the current iteration of the loop and proceeds to the next one.