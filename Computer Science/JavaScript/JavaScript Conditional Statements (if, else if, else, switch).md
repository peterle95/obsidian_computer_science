---
memory: done
tags:
  - mastered
language:
  - JavaScript
review-date: ""
last-reviewed: 2025-07-24
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 3
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
Conditional statements are fundamental to all programming languages, including JavaScript, because they enable programs to make decisions and execute different blocks of code based on whether certain conditions are true or false. The fundamental problem they solve is how to introduce "intelligence" or "logic" into a program, allowing it to respond dynamically to varying inputs, data states, or user interactions. Without conditional statements, a program would execute the same sequence of instructions every time, regardless of circumstances, severely limiting its utility. They are crucial for creating interactive applications, handling errors, validating input, controlling program flow, and implementing algorithms that require branching logic. In JavaScript, this is vital for building dynamic web pages, server-side applications with Node.js, and complex interactive user interfaces.

# **Core Explanation:**

---

Conditional statements in JavaScript allow you to execute different blocks of code depending on the evaluation of a condition. The primary conditional statements are `if`, `else if`, `else`, and `switch`.

**1. `if` Statement:**
The `if` statement is the most basic conditional statement. It executes a block of code if a specified condition evaluates to `true`.

* **Syntax:**
 ```javascript
 if (condition) {
 // code to be executed if condition is true
 }
 ```
* **How it works:** The `condition` inside the parentheses is evaluated. If it's `true` (or a "truthy" value), the code inside the curly braces `{}` is executed. If it's `false` (or a "falsy" value), the code block is skipped.

**2. `else` Statement:**
The `else` statement is used in conjunction with an `if` statement to execute a block of code if the `if` condition evaluates to `false`.

* **Syntax:**
 ```javascript
 if (condition) {
 // code to be executed if condition is true
 } else {
 // code to be executed if condition is false
 }
 ```
* **How it works:** If the `if` condition is `true`, its block runs. Otherwise (if the `if` condition is `false`), the `else` block runs. Only one of the two blocks will execute.

**3. `else if` Statement:**
The `else if` statement allows you to test multiple conditions in sequence. It's used to specify a new condition to test if the previous `if` or `else if` condition(s) evaluated to `false`. You can have multiple `else if` blocks.

* **Syntax:**
 ```javascript
 if (condition1) {
 // code if condition1 is true
 } else if (condition2) {
 // code if condition1 is false AND condition2 is true
 } else {
 // code if all preceding conditions are false
 }
 ```
* **How it works:** Conditions are evaluated from top to bottom. The first `if` or `else if` condition that evaluates to `true` has its corresponding code block executed, and then the rest of the `else if` and `else` chain is skipped. If none of the `if` or `else if` conditions are true, the optional `else` block is executed.

**4. `switch` Statement:**
The `switch` statement is an alternative to long `if-else if` chains when you need to perform different actions based on different values of a single variable or expression. It compares the value of an expression against multiple `case` values.

* **Syntax:**
 ```javascript
 switch (expression) {
 case value1:
 // code to execute if expression matches value1
 break; // Important: exits the switch block
 case value2:
 // code to execute if expression matches value2
 break;
 default:
 // code to execute if expression doesn't match any case
 }
 ```
* **How it works:** The `expression` is evaluated once. Its value is then compared, strictly (`/===`), with the values of each `case`.
 * If a match is found, the code associated with that `case` is executed.
 * The `break` keyword is crucial; it terminates the `switch` statement. Without `break`, execution "falls through" to the next `case`'s code (this is known as "fall-through behavior," often undesirable).
 * The `default` case is optional and executes if no `case` matches the `expression`'s value.

# **Related Concepts:**

---
* **Boolean Logic/Operators:** Conditional statements heavily rely on boolean values (`true` or `false`) and boolean operators (`&&` AND, `||` OR, `!` NOT) to construct complex conditions. The outcome of these operations determines which code path is taken.
* **Comparison Operators:** These operators (`/==`, `/===`, `!=`, `!==`, `>`, `<`, `>=`, `<=`) are used within the conditions of `if`, `else if`, and `switch` statements to compare values and produce a boolean result.
* **Ternary Operator (Conditional Operator):** This is a shorthand conditional operator (`condition ? expressionIfTrue : expressionIfFalse`). It's a concise way to write simple `if-else` statements that return a value. It differs by being an expression that evaluates to a value, rather than a statement that executes a block of code.
* **Control Flow:** Conditional statements are core components of a program's control flow, determining the order in which instructions are executed. Other control flow structures include loops (for, while) and function calls.
* **Type Coercion (for `if` and `else if` conditions):** In JavaScript, conditions in `if` and `else if` statements implicitly convert non-boolean values to booleans (`truthy` and `falsy`). `0`, `null`, `undefined`, `NaN`, `""` (empty string), and `false` are "falsy"; all other values are "truthy." This is a key difference from `switch` which uses strict comparison (`/===`).

# **Examples:**

---
```javascript
// Example 1: if, else if, else
let temperature = 25;

if (temperature > 30) {
 // This block executes if temperature is strictly greater than 30
 console.log("It's a very hot day!");
} else if (temperature >= 20) {
 // This block executes if the first condition (temp > 30) was false,
 // AND temperature is greater than or equal to 20.
 console.log("It's a pleasant day.");
} else {
 // This block executes if none of the above conditions were true
 console.log("It's a bit chilly.");
}

// Example 2: switch statement for day of the week
let dayOfWeek = 3; // 0 for Sunday, 1 for Monday, etc.

switch (dayOfWeek) {
 case 0:
 // This case executes if dayOfWeek is exactly 0
 console.log("It's Sunday.");
 break; // Stop executing further cases
 case 1:
 // This case executes if dayOfWeek is exactly 1
 console.log("It's Monday.");
 break;
 case 2:
 // This case executes if dayOfWeek is exactly 2
 console.log("It's Tuesday.");
 break;
 case 3:
 // This case executes if dayOfWeek is exactly 3
 console.log("It's Wednesday.");
 break;
 case 4:
 // This case executes if dayOfWeek is exactly 4
 console.log("It's Thursday.");
 break;
 case 5:
 // This case executes if dayOfWeek is exactly 5
 console.log("It's Friday.");
 break;
 case 6:
 // This case executes if dayOfWeek is exactly 6
 console.log("It's Saturday.");
 break;
 default:
 // This block executes if dayOfWeek does not match any of the above cases
 console.log("Invalid day number.");
}

// Example 3: Truthy and Falsy values in if statements
let userName = ""; // An empty string is a "falsy" value

if (userName) {
 // This block will NOT execute because "" is falsy
 console.log("Welcome, " + userName + "!");
} else {
 // This block WILL execute
 console.log("Please enter your username.");
}

let userAge = 0; // The number 0 is a "falsy" value

if (userAge) {
 // This block will NOT execute because 0 is falsy
 console.log("User is " + userAge + " years old.");
} else {
 // This block WILL execute
 console.log("Age not provided or is 0.");
}

let isValid = true; // A boolean true is a "truthy" value

if (isValid) {
 // This block WILL execute
 console.log("The data is valid.");
} else {
 console.log("The data is invalid.");
}
```

# **Flashcards:**

---
What is the purpose of `if` and `else` statements in JavaScript?;; To execute different blocks of code based on whether a condition is true or false.

When would you use a `switch` statement instead of `if-else if`?;; When you need to compare a single expression against multiple distinct possible values, providing a cleaner and often more readable structure.

What is the importance of the `break` keyword in a `switch` statement?;; It prevents "fall-through" behavior, ensuring that only the code block for the matched `case` is executed, and then the `switch` statement is exited.