---
memory: to_finish
tags:
  - learning
language:
  - JavaScript
review-date: 2025-11-25
last-reviewed: 2025-10-16
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-09-18
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
# Purpose/Why:
---

JavaScript is a ==**single-threaded** language==, meaning it can only execute one command at a time.

The fundamental ==problem arises with long-running operations, such as fetching data from a server, reading a file, or complex calculations.== <mark style="background: #BBFABBA6;">If JavaScript were purely **synchronous**, a long task would **block** the entire program. In a web browser, this would freeze the user interface (UI), making the page unresponsive to clicks, scrolls, or any user interaction. This creates a poor user experience.</mark> 😵

==**Asynchronous** programming solves this by allowing the program to initiate a long-running task and then continue with other operations without waiting for the first task to complete.== Once the task finishes, its result is handled. This non-blocking model is essential for creating responsive, modern applications that can efficiently handle I/O-bound operations (like network requests) without freezing up.

# Core Explanation:
---

The key difference between synchronous and asynchronous programming lies in <mark style="background: #D2B3FFA6;">how tasks are executed.</mark>

## Synchronous (Sync)

**Synchronous** means "at the same time" or "in sequence." In programming, this translates to a **blocking** execution model.

- **Definition**: Code is executed line by line, one after another.<mark style="background: #D2B3FFA6;"> Each task must finish completely</mark> before the next one can begin.
- **Analogy**: A single-lane checkout at a grocery store. The cashier must fully process one customer's order before starting with the next person in line. 🛒

- **Characteristics**:
 - **Blocking**: Halts the program until a task is done.
 - **Predictable**: The order of operations is easy to follow.
 - **Single-threaded**: Executes on the main thread.

## Asynchronous (Async)

**Asynchronous** means "<mark style="background: #ABF7F7A6;">not at the same time</mark>." In programming, this is a **non-blocking** execution model.

- **Definition**: Allows the program to <mark style="background: #ABF7F7A6;">start a long-running task (like an API call) and move on to other tasks immediately.</mark> The <mark style="background: #ABF7F7A6;">program doesn't wait for the first task to finish</mark>. <mark style="background: #ABF7F7A6;">When the result is ready, it's handled</mark> by a callback function, a Promise, or `async/await`.

- **Analogy**: Ordering at a coffee shop with a buzzer. You place your order (initiate the task) and receive a buzzer. You are then free to find a table or chat with a friend (the program does other things). When your coffee is ready (the task is complete), the buzzer vibrates (the callback/promise resolves), and you go collect your order (handle the result). ☕

- **How it works**: JavaScript uses an **Event Loop** concurrency model. When an async operation is called, it's handed off to a separate engine (like the browser's Web APIs). Once that operation is complete, a callback function is placed in a task queue. The Event Loop continuously checks if the main call stack is empty; if it is, it pushes the first task from the queue onto the stack for execution.

# Related Concepts:

---
- **Single-Threaded**: JavaScript's nature of having only one call stack. This limitation is precisely why the asynchronous model is so crucial to avoid blocking that single thread.

- **Blocking vs. Non-Blocking**: These terms describe the effect of a task on program flow. **Synchronous** operations are **blocking**. **Asynchronous** operations are **non-blocking**.

- **Event Loop, Call Stack, & Task Queue**: These are the core components of JavaScript's concurrency model. The **Call Stack** tracks function execution. The **Task Queue** holds results from completed async operations. The **Event Loop**'s job is to move tasks from the queue to the stack when the stack is empty. This system is what enables non-blocking behavior.

- **Callbacks, Promises, and `async/await`**: These are the tools and patterns used to _manage_ asynchronous operations.

 - **[[JavaScript Callback Functions]]**: The original pattern; a function passed to another function to be executed later.

 - **[[JavaScript Promises (then, catch, finally)]]**: An object representing the eventual completion (or failure) of an async operation.

 - **[[JavaScript Async⧸Await (ES8)]]**: Syntactic sugar over Promises to make async code look and feel synchronous.

# Examples:

---
```js
/*
---
- Synchronous Example
---
-
*/
// The code executes top-to-bottom, one line at a time.
console.log("Sync: First task - Making breakfast.");

// Imagine this is a heavy, blocking task.
// The program will pause here and be unresponsive until the loop finishes.
console.log("Sync: Second task - Toasting bread (takes a while)...");
for (let i = 0; i < 2e9; i++) {
 // Simulating a long-running, blocking operation.
}
console.log("Sync: Bread is toasted!");

// This line CANNOT run until the loop above is 100% complete.
console.log("Sync: Third task - Adding butter.");

console.log("
---
");

 // Separator for clarity

/*
---
- Asynchronous Example
---
-
*/
// The code still starts top-to-bottom, but some tasks are non-blocking.
console.log("Async: First task - Start making breakfast.");

// setTimeout is an asynchronous function.
// We are telling the JavaScript environment: "Please run this function, but wait at least 2000ms.
// In the meantime, DON'T WAIT. Continue executing the rest of the code."
setTimeout( => {
 // This code runs LATER, after the delay. It doesn't block the main thread.
 console.log("Async: Second task - Bread is finally toasted!");
}, 2000); // 2000 milliseconds = 2 seconds

// This line executes IMMEDIATELY after setTimeout is called.
// It does NOT wait for the 2-second timer to finish.
console.log("Async: Third task - Adding butter while waiting for toast.");
```

# Flashcards:

---
What is the main characteristic of synchronous programming?;;**Blocking**: Tasks are executed sequentially, and each task must complete before the next one begins

What is the main characteristic of asynchronous programming?;;**Non-blocking**: Long-running tasks can be initiated, and the program can continue with other operations without waiting for the initial task to complete

Why is asynchronous programming crucial for JavaScript in a browser environment?;;Because JavaScript is single-threaded. Synchronous (blocking) operations would freeze the user interface, making the webpage unresponsive Asynchronous operations prevent this

In a simple analogy, if synchronous code is a single-file line, what is asynchronous code like?;;Ordering food with a buzzer. You place your order (start a task) and can do other things until the buzzer goes off (the task completes and notifies you).

What is the name of the core JavaScript mechanism that enables asynchronous behavior?;;The **Event Loop** (along with the Call Stack and Task Queue).

Name three patterns/tools used to handle asynchronous operations in JavaScript.;;Callbacks, Promises, and `async/await`