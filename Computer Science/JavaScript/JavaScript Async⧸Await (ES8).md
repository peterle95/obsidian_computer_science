---
tags:
  - learning
language:
  - JavaScript
review-date: 2025-11-25
last-reviewed: 2025-10-16
scheda: done
visit-count: 4
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-09-17
memory: done
cssclasses:
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
# Purpose/Why:
---

`async/await` was introduced in ES8 (ECMAScript 2017) to solve the problem of managing complex asynchronous operations in JavaScript.

==Before `async/await`, developers primarily used **[[JavaScript Callback Functions]]** and later **[[JavaScript Promises (then, catch, finally)]]** (`.then()` and `.catch()`).== <mark style="background: #ADCCFFA6;">Callbacks</mark> often led to a <mark style="background: #ADCCFFA6;">difficult-to-read</mark>, deeply nested structure known as <mark style="background: #ADCCFFA6;">"callback hell" or the "pyramid of doom."</mark> <mark style="background: #BBFABBA6;">Promises improved this by allowing chaining, but the syntax could still be verbose and less intuitive than standard synchronous code.</mark>

==`async/await` provides **syntactic sugar** on top of Promises==. Its primary purpose is to let us write ==asynchronous code that **looks and feels synchronous**==. This makes the code significantly ==more readable, easier to debug, and simpler to reason about, especially when dealing with multiple sequential asynchronous tasks== (e.g., fetching user data, then fetching their posts, then fetching comments for a post). It's crucial for modern web development where non-blocking operations like API calls, file I/O, and database queries are commonplace.

# Core Explanation:
---

`async/await` is a modern feature for handling asynchronous operations in JavaScript. It consists of two keywords: `async` and `await`.

### `async`
* The `async` keyword is placed before a function declaration.
* It automatically ==transforms the function into one that returns a **Promise**==.
* If the function explicitly returns a value (e.g., `return 42`), the Promise it returns will resolve with that value.
* If the function throws an error, the Promise it returns will be rejected with that error.

### `await`
* The `await` keyword can ==**only be used inside an `async` function**==.
* It is ==placed in front of any Promise-based expression (like a `fetch()` call or another `async` function).==
* It ==**pauses the execution of the `async` function** at that line until the Promise settles== (is either resolved or rejected).
* If the Promise resolves, `await` returns the resolved value.
* If the Promise is rejected, `await` throws the rejection value as an error, which can be caught using a standard synchronous `try...catch` block.

==Essentially, `async/await` allows you to "wait" for an asynchronous task to complete and get its result without blocking the main JavaScript thread, all while using a clean, linear syntax.==

# **Memory Palace**
---
## **1. Chosen Location / Room**
- Living Room
## **2. Loci (Memory Spots)**
_List 1–3 physical “spots” in that location that you will attach information to._
- Spot 1:  
- Spot 2:  
- Spot 3:  
## **3. Encoded Imagery / Story (Visual OR Non-Visual)**
_Describe the mnemonic you attach to each spot. This can be visual, verbal, symbolic, conceptual, or sensory._
- Spot 1 mnemonic:  When you approach the gas machine, it suddenly _waits_ before turning on — representing the idea that `await` pauses execution until something finishes.

- Spot 2 mnemonic:  Opening the fridge requires “permission” — like marking a function `async` before you’re allowed to use `await` inside.

- Spot 3 mnemonic:  Water flows only _after_ you turn the handle, symbolizing how Promises resolve only after an asynchronous task completes.

## **4. Retrieval Path**
Enter kitchen → gas machine (await pauses execution) → fridge (async enables awaiting) → sink (Promise resolves and flows).

# Related Concepts:
---

* **Promises**: This is the foundational concept. `async/await` is built directly on top of Promises. An `async` function always returns a Promise, and the `await` keyword is designed to wait for a Promise to resolve. You can't fully understand `async/await` without understanding Promises. The main difference is **syntax**: `async/await` provides a cleaner, more synchronous-looking way to work with the same underlying Promise-based logic.

* **Callbacks**: The original mechanism for handling asynchronous tasks in JavaScript. A callback is a function passed as an argument to another function, to be executed later. `async/await` is a modern alternative that avoids the major drawback of callbacks: "callback hell," where nested callbacks become unmanageable.

* **Event Loop**: The core concurrency model in JavaScript. The event loop allows JavaScript to perform non-blocking I/O operations despite being single-threaded. When an `async` function `await`s a result, it yields control back to the event loop, allowing other code to run. Once the awaited Promise settles, the function's execution is resumed. `async/await` doesn't change this model; it just provides a more elegant way to interact with it.

# Examples:
---
```javascript
// This function simulates a network request. It returns a Promise
// that resolves after a specified delay.
function fetchData(data, delay) {
  // We return a new Promise. The 'resolve' function will be called when our async task is done.
  return new Promise(resolve => {
    setTimeout(() => {
      console.log(`Finished fetching: ${data}`);
      resolve({ fetched: data }); // Resolve the promise with the data
    }, delay);
  });
}

// ---- The Traditional Promise .then() chain ----
// This is how we would handle sequential async tasks before async/await.
function getPostDataWithPromises() {
  console.log("Starting Promise chain...");
  fetchData("User Data", 1000)
    .then(userData => {
      // Once the first promise resolves, we start the next one.
      console.log("Got user data, now fetching posts...");
      return fetchData("User Posts", 1500); // This returns another promise
    })
    .then(postData => {
      // Once the second promise resolves...
      console.log("Got posts, now fetching comments...");
      return fetchData("Post Comments", 800);
    })
    .then(commentData => {
      // Final step.
      console.log("All data fetched with Promises!");
      console.log("---");
    })
    .catch(error => {
      // A single .catch() can handle errors from any part of the chain.
      console.error("An error occurred in the Promise chain:", error);
    });
}

// getPostDataWithPromises(); // Uncomment to run the promise example.


// ---- The Modern async/await approach ----
// Notice the 'async' keyword. This tells JavaScript this function will contain asynchronous operations.
async function getPostDataWithAsyncAwait() {
  try {
    // We wrap our asynchronous code in a try...catch block for error handling,
    // just like we would with synchronous code.
    console.log("Starting async/await execution...");

    // The 'await' keyword pauses the function execution here.
    // It waits for the fetchData promise to resolve, then assigns the resolved value to 'userData'.
    // The code looks like it's stopping, but the JS event loop is free to do other work.
    const userData = await fetchData("User Data", 1000);
    console.log("Got user data, now fetching posts...");

    // Execution resumes, and we await the next promise. The code is clean and linear.
    const postData = await fetchData("User Posts", 1500);
    console.log("Got posts, now fetching comments...");

    // Awaiting the final piece of data.
    const commentData = await fetchData("Post Comments", 800);

    console.log("All data fetched with async/await!");

  } catch (error) {
    // If any of the 'await'ed promises are rejected, the 'catch' block will execute.
    // This is a much more familiar error handling pattern for most developers.
    console.error("An error occurred during async/await execution:", error);
  }
}

// Call the async function to start the process.
getPostDataWithAsyncAwait();
```

# Flashcards:
---

What does the `async` keyword do when placed before a function?;;It implicitly makes the function return a Promise. If the function returns a value, the Promise resolves with that value. If it throws an error, the Promise is rejected.


What does the `await` keyword do, and where can it be used?;;`await` pauses the execution of its containing function until a Promise is settled. It can only be used inside an `async` function.


How is error handling managed with `async/await`?;;Using standard synchronous `try...catch` blocks. If an awaited Promise is rejected, it throws an error that can be caught by a `catch` block.


What does an `async` function ALWAYS return?;;A Promise.

`async/await` is syntactic sugar for what underlying JavaScript concept?;;Promises. It provides a cleaner, more readable syntax for consuming Promises.


What primary problem in asynchronous JavaScript does `async/await` solve?;;It solves the problem of "callback hell" and cleans up complex `.then()` chains, making asynchronous code look and behave more like synchronous code, thus improving readability and maintainability.