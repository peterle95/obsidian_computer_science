---
tags:
  - to_learn
language:
  - JavaScript
review-date:
last-reviewed: ""
scheda: done
visit-count: 0
confidence-level: 1
consecutive-correct: 0
last-struggle-date: ""
cssclasses:
memory: to_finish
palace:
palace-room:
locus:
palace-order:
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

`new Function` lets you create a function from a string at runtime, which is useful when function code is only known dynamically (for example, received from a server or built from a template in complex web apps). This enables dynamic compilation/execution of code but comes with security and maintainability trade‑offs.​

# **Core Explanation:**

---

The basic syntax is: `let func = new Function([arg1, arg2, ...argN], functionBody);` where parameter names and body are strings. In practice this is usually written as string arguments: `new Function('a', 'b', 'return a + b')`, or `new Function('a,b', 'return a + b')` (with or without spaces).​

Unlike normal function declarations or expressions, `new Function` builds the function **from** its string body at runtime instead of from code written directly in the script. You can thus do things like `let str = "...some code from server..."; let f = new Function(str); f();` to execute dynamically supplied code.​

A crucial detail is closure behavior: functions created with `new Function` have `[[Environment]]` set to the global lexical environment, not the surrounding local one. That means they cannot see local outer variables, only globals, so code like `let value = "test"; let f = new Function('alert(value)');` inside another function will throw `value is not defined` unless `value` is global.​

This design avoids problems with minifiers, which rename local variables (e.g., `userName` → `a`) and would otherwise break dynamically created functions if they depended on local names. Architecturally, it forces you to pass data via explicit parameters instead of accidentally capturing outer state, which is more robust for dynamically compiled code.​

# **Related Concepts:**

---

- Regular function declarations/expressions: Normal functions are written directly in code, created at parse time, and capture their lexical environment (closures) so they can access outer local variables.[[javascript](https://javascript.info/new-function)]​
    
- Closures and lexical environment: Standard JS functions store a `[[Environment]]` reference to the scope where they were created; `new Function` is the notable exception because it always points to the global environment.[[javascript](https://javascript.info/new-function)]​
    
- Minification: Minifiers rename local variables to short names; since `new Function` does not rely on local outer variables, its behavior remains stable after minification.[[javascript](https://javascript.info/new-function)]​
    
- `eval`: Like `new Function`, `eval` can execute code from a string, but it runs in the current scope and can access local variables, making its behavior different and often more dangerous or confusing.[[javascript](https://javascript.info/new-function)]​
    

# **Examples:**

---


```js
// Example 1: Simple sum function created with new Function
// Here we build a function from a string: two parameters 'a' and 'b' and a body that returns a + b.[page:1]
const sum = new Function('a', 'b', 'return a + b'); // parameters and body are strings[page:1]

// This behaves like a normal function call, returning 3.[page:1]
console.log(sum(1, 2)); // 3[page:1]

// Example 2: Function with only a body (no parameters)
// Here we create a function that simply shows a message; its body is a single string.[page:1]
const sayHi = new Function('console.log("Hello from new Function")'); // only body string[page:1]

// Calling sayHi executes the dynamically built function.[page:1]
sayHi(); // Hello from new Function[page:1]

// Example 3: Receiving code from a "server" dynamically
// Imagine this string is loaded from a server or built from a template at runtime.[page:1]
const codeFromServer = 'return `Server says: ${msg.toUpperCase()}`;';

// We create a new function taking 'msg' as an argument, and using the body from the server string.[page:1]
const serverFunc = new Function('msg', codeFromServer); // dynamic body[page:1]

// Now we can call it like any other function, passing arguments explicitly.[page:1]
console.log(serverFunc('hello')); // "Server says: HELLO"[page:1]


// Example 4: Demonstrating lack of access to local outer variables
function getFuncNewFunction() {
  // Local variable that a normal closure would see.[page:1]
  const value = 'local test';

  // new Function creates in the global environment, not in this local scope.[page:1]
  const func = new Function('console.log(value)'); // tries to use 'value'[page:1]

  return func;
}

// This will throw ReferenceError in strict situations because 'value' is not global.[page:1]
try {
  getFuncNewFunction()(); // value is not defined[page:1]
} catch (e) {
  console.log('Error as expected:', e.message);
}


// Example 5: Regular closure contrast (works with outer variable)
function getFuncRegular() {
  // Local variable that will be closed over.[page:1]
  const value = 'local test';

  // Normal function expression captures the lexical environment of getFuncRegular.[page:1]
  const func = function () {
    console.log(value); // closes over value[page:1]
  };

  return func;
}

// This prints "local test" because regular functions form closures.[page:1]
getFuncRegular()(); // "local test"[page:1]


// Example 6: Proper way to pass data into new Function – via parameters
// We define a function factory that returns a new Function using explicit parameters.[page:1]
function makeMultiplier(factor) {
  // 'x' is parameter name, body uses x and factor; factor is interpolated into the string now.[page:1]
  // Note: factor is baked into the string at creation time, not captured as a closure variable.[page:1]
  return new Function('x', `return x * ${factor};`);
}

// Build a "times 3" function dynamically.
const times3 = makeMultiplier(3);

// This still works correctly and does not depend on outer scope at call time.[page:1]
console.log(times3(10)); // 30[page:1]

```
# **Flashcards:**

---

What is the main purpose of new Function in JavaScript?;;To create a function from a string at runtime when the function code is only known dynamically

Write the basic syntax of new Function with parameters and body.;;let func = new Function('a', 'b', 'return a + b'); where parameter names and body are strings

How does the Environment of a new Function-created function differ from a regular function?;;new Function functions reference the global lexical environment, while regular functions capture their surrounding local lexical environment (closures)

Why doesn’t new Function have access to outer local variables?;;Because its Environment is intentionally set to the global scope to avoid issues with minification and accidental dependence on local names

What is the recommended way to pass data into a function created with new Function?;;Pass values explicitly as arguments and use parameter names in the function’s string body

Give a practical use case where new Function is appropriate.;;Dynamically compiling and executing function code received from a server or generated from templates in complex web applications​