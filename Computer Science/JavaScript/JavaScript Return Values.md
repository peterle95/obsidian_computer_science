---
memory: to_finish
tags:
  - learned
language:
  - JavaScript
review-date: ""
last-reviewed: 2025-09-03
scheda: done
visit-count: 2
confidence-level: 1.5
consecutive-correct: 2
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

The concept of **return values** in JavaScript functions solves the problem of **passing data *out* of a function** back to the part of the code that called it. While parameters allow data to flow *into* a function, return values enable a function to produce a result or communicate the outcome of its operations. This is critical for:
* **Enabling calculations and transformations**: Functions can compute values (e.g., `add(2, 3)` returns `5`).
* **Facilitating function composition**: The output of one function can become the input of another, allowing for complex data processing pipelines.
* **Controlling program flow**: The presence or value of a return can dictate subsequent actions in the calling code.
* **Creating reusable components**: Functions become like "black boxes" that take inputs and produce predictable outputs, making them highly versatile.

Without return values, functions would only be able to perform side effects (like printing to the console or modifying global variables), severely limiting their utility and making code harder to manage and test.

# **Core Explanation:**

---
In JavaScript, the `return` keyword is used inside a function to **send a value back** to the caller. When the JavaScript interpreter encounters a `return` statement within a function, two things happen:

1. **Function Termination**: The execution of the function immediately stops at that point. Any code located after the `return` statement in the function body will not be executed.
2. **Value Delivery**: The specified value (or `undefined` if no value is explicitly given) is returned to the place where the function was called. This returned value can then be captured in a variable, used in an expression, or passed as an argument to another function.

**Key aspects of return values:**
* **Any Data Type**: A function can return any valid JavaScript data type: primitives (numbers, strings, booleans, null, undefined), or complex types (objects, arrays, other functions).
* **Implicit `undefined`**: If a function does not have an explicit `return` statement, or if it has a `return;` statement without a value, the function will implicitly return `undefined`.
* **Capturing the Value**: The value returned by a function can be assigned to a variable, used directly in an expression, or passed as an argument to another function.

Return values are fundamental for making functions truly useful beyond just performing actions; they allow functions to compute and provide data back to the rest of the program.

# **Related Concepts:**

---
* **JavaScript Functions**: Return values are an integral part of function definition and execution, serving as the primary mechanism for a function to output data.
* **JavaScript Function Parameters and Arguments**: While parameters/arguments are used to pass data *into* a function (inputs), return values are used to pass data *out* of a function (outputs). Together, they define the input-output interface of a function.
* **Variables**: Returned values are frequently assigned to variables, which then hold the result of the function's execution, allowing that result to be used later in the program.
* **Expressions**: A function call that returns a value behaves like an expression. For example, `add(2, 3)` is an expression that evaluates to `5` if the `add` function returns the sum. This allows function calls to be embedded within larger expressions.
* **Control Flow**: The `return` statement directly impacts control flow by immediately exiting the function. This can be used to prematurely terminate a function's execution based on certain conditions, which is crucial for handling different scenarios within a function.
* **Function Chaining/Composition**: The ability of functions to return values is essential for chaining or composing functions, where the output of one function becomes the input for the next, creating a sequence of operations.

# **Examples:**

---
```javascript
//
---
1. Basic Function with a Return Value
---
// A function that calculates the area of a rectangle and returns the result
function calculateRectangleArea(length, width) {
 let area = length * width; // Calculate the area
 return area; // Return the calculated area
}

// Call the function and store the returned value in a variable
let livingRoomArea = calculateRectangleArea(10, 5);
console.log("Living Room Area:", livingRoomArea); // Output: Living Room Area: 50

// Use the returned value directly in an expression
let totalArea = calculateRectangleArea(8, 4) + calculateRectangleArea(6, 3);
console.log("Total Area:", totalArea); // Output: Total Area: 50 (32 + 18)

//
---
2. Returning Different Data Types
---
// Function returning a string
function getFullName(firstName, lastName) {
 return `${firstName} ${lastName}`; // Returns a concatenated string
}
let fullName = getFullName("John", "Doe");
console.log("Full Name:", fullName); // Output: Full Name: John Doe

// Function returning a boolean
function isEven(number) {
 return number % 2 === 0; // Returns true if even, false if odd
}
console.log("Is 4 even?", isEven(4)); // Output: Is 4 even? true
console.log("Is 7 even?", isEven(7)); // Output: Is 7 even? false

// Function returning an array
function createList(item1, item2) {
 return [item1, item2]; // Returns a new array
}
let myList = createList("Apple", "Banana");
console.log("My List:", myList); // Output: My List: [ 'Apple', 'Banana' ]

// Function returning an object
function createUser(name, email) {
 return { userName: name, userEmail: email }; // Returns a new object
}
let newUser = createUser("Alice", "alice@example.com");
console.log("New User:", newUser); // Output: New User: { userName: 'Alice', userEmail: 'alice@example.com' }

//
---
3. Implicit Undefined Return
---
// A function without an explicit return statement
function logMessage(message) {
 console.log(message); // Performs an action (side effect), but doesn't return data
}
let resultOfLog = logMessage("This is a test message."); // Output: This is a test message.
console.log("Result of logMessage:", resultOfLog); // Output: Result of logMessage: undefined
// Even though it printed, the function itself returned no specific value.

//
---
4. Return as Immediate Exit
---
// A function demonstrating that code after 'return' is not executed
function checkAge(age) {
 if (age < 18) {
 return "Too young"; // If age is less than 18, this line returns and the function exits
 console.log("This line will never be reached if age < 18."); // This code is unreachable
 }
 return "Adult"; // This line is reached only if age is 18 or greater
}

console.log(checkAge(15)); // Output: Too young
console.log(checkAge(25)); // Output: Adult

// Another example: early exit for validation
function divide(a, b) {
 if (b === 0) {
 console.error("Error: Cannot divide by zero!");
 return null; // Return null (or throw an error) to indicate failure
 }
 return a / b;
}

console.log(divide(10, 2)); // Output: 5
console.log(divide(5, 0)); // Output: Error: Cannot divide by zero! (and then) null
````

# **Flashcards:**

---
What does the return keyword do in a JavaScript function?;;It immediately stops the function's execution and sends a value back to the caller.

What happens if a JavaScript function does not have a return statement?;;It implicitly returns undefined.

Can a JavaScript function return an object or an array?;;Yes, a function can return any valid JavaScript data type, including primitive values, objects, and arrays.