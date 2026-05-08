---
memory: done
tags:
  - mastered
language:
  - JavaScript
review-date: ""
last-reviewed: 2025-08-31
scheda: done
visit-count: 4
confidence-level: 2.5
consecutive-correct: 1
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
Functions in JavaScript (and programming in general) solve the fundamental problem of **code reusability and organization**. Instead of writing the same lines of code multiple times, functions allow you to encapsulate a specific task or behavior into a reusable block. This promotes **modularity**, breaking down complex problems into smaller, manageable pieces, and enhances **maintainability**, making code easier to understand, debug, and modify. They are crucial for creating structured, efficient, and scalable applications, preventing redundancy, and improving readability by abstracting away complex logic behind simple function calls.

# **Core Explanation:**
---
A **function** in JavaScript is a block of organized, reusable code that performs a specific task. Functions are defined once and can be executed, or "called," multiple times throughout your program.

==To make the code clean and easy to understand, it’s recommended to use mainly local variables and parameters in the function, not outer variables.==

**Key characteristics and how they work:**
* **Definition**: A function is declared using the `function` keyword, followed by a name, a list of parameters (inputs) enclosed in parentheses, and a block of code (the function body) enclosed in curly braces `{}`.
* **Parameters and Arguments**:
    * **Parameters** are variables listed inside the parentheses in the function definition. They act as placeholders for values that will be passed into the function.
    * **Arguments** are the actual values supplied to the function when it is called. These arguments are mapped to the function's parameters.
* **Execution (Calling/Invoking)**: A function's code inside its body is executed only when the function is explicitly "called" or "invoked" by its name, followed by parentheses (e.g., `myFunction()`).
* **Return Values**: Functions can optionally return a value using the `return` keyword. When `return` is encountered, the function stops execution, and the specified value is sent back to the caller. If no `return` statement is present or if `return;` is used without a value, the function implicitly returns `undefined`.
* **Scope**: Variables declared inside a function are local to that function and cannot be accessed from outside it (local scope), preventing naming conflicts and accidental modification of data.

Functions are the building blocks of JavaScript applications, enabling complex behaviors to be composed from simpler, well-defined operations.

# **Related Concepts:**
---
* **Variables**: Functions often interact with variables. They can take variables as arguments, declare local variables, and return values that can be assigned to variables. Understanding variable scope (global vs. local) is crucial for managing data within and outside functions.
* **Control Flow**: Functions directly influence the control flow of a program. When a function is called, execution jumps to the function body, executes its code, and then returns to the point where it was called. Conditional statements (`if/else`) and loops (`for`, `while`) are often used *within* function bodies to control their internal logic.
* **Data Types (Primitives & Objects)**: Functions operate on data, which can be of any JavaScript data type. Parameters can accept numbers, strings, booleans, objects, arrays, and even other functions. The type of data returned by a function can also be any valid JavaScript type.
* **Modularity**: This is a direct benefit of using functions. Functions promote modularity by encapsulating related code into independent, reusable units. This makes code easier to test, debug, and maintain.
* **Abstraction**: Functions provide a level of abstraction. You can use a function without needing to know the intricate details of its internal implementation, focusing instead on what it *does* rather than *how* it does it. This simplifies the usage of complex operations.

# **Examples:**
---
```javascript
// --- 1. Basic Function Declaration and Calling ---

// Define a simple function named 'greet'
function greet() {
  console.log("Hello, JavaScript!"); // This code runs when the function is called
}

// Call the 'greet' function to execute its code
greet(); // Output: Hello, JavaScript!

// Call it again to demonstrate reusability
greet(); // Output: Hello, JavaScript!


// --- 2. Function with Parameters (Inputs) ---

// Define a function 'greetPerson' that takes one parameter: 'name'
function greetPerson(name) {
  console.log(`Hello, ${name}!`); // The 'name' parameter is used inside the function
}

// Call 'greetPerson' and pass "Alice" as an argument for the 'name' parameter
greetPerson("Alice"); // Output: Hello, Alice!

// Call it again with a different argument
greetPerson("Bob"); // Output: Hello, Bob!


// Define a function 'addNumbers' that takes two parameters: 'num1' and 'num2'
function addNumbers(num1, num2) {
  let sum = num1 + num2; // Perform an operation using the parameters
  console.log(`The sum is: ${sum}`);
}

// Call 'addNumbers' with two arguments
addNumbers(5, 3); // Output: The sum is: 8
addNumbers(10, -2); // Output: The sum is: 8


// --- 3. Function with a Return Value ---

// Define a function 'multiply' that takes two parameters and returns their product
function multiply(a, b) {
  let product = a * b; // Calculate the product
  return product; // The 'return' keyword sends 'product' back to where the function was called
}

// Call 'multiply' and store its returned value in a variable
let result1 = multiply(4, 5);
console.log(`Result of multiplication: ${result1}`); // Output: Result of multiplication: 20

// Use the returned value directly in another operation
let finalResult = multiply(10, 2) + 5;
console.log(`Final result: ${finalResult}`); // Output: Final result: 25 (20 + 5)

// A function without an explicit return statement returns 'undefined'
function doNothing() {
  // No return statement
}
let nothing = doNothing();
console.log(nothing); // Output: undefined
````

# **Flashcards:**

---

What is a JavaScript function?;;A reusable block of code that performs a specific task.

How do you execute (call) a JavaScript function?;;By writing its name followed by parentheses, e.g., myFunction().

What is the purpose of the return keyword in a function?;;To send a value back to the caller and stop the function's execution.