---
tags:
  - learning
language:
  - JavaScript
review-date: 2025-11-25
last-reviewed: 2025-10-17
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-09-18
cssclasses:
palace-room:
palace:
locus:
palace-order: "2"
memory: done
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

JavaScript code cannot run on its own. I==t needs a special program called a **runtime environment** that can interpret and execute it.== Understanding the different ways to run JavaScript is the most fundamental step in using the language.

The primary application is twofold:

1. **Client-Side (Browser):** To create dynamic and interactive user experiences on websites.
    
2. **Server-Side (Node.js):** To build fast and scalable backend services, APIs, and command-line tools.
    
Knowing the execution environment is critical because it dictates what your code can and cannot do. For example, ==browser-based JavaScript can manipulate a webpage's content, while server-side Node.js can read and write files on a computer==.

# Core Explanation:
---

JavaScript is an interpreted language that requires a **runtime environment** to execute. This environment consists of a **JavaScript Engine** (which reads and runs the code) and a set of specific **APIs** that allow the code to interact with the world outside of the language itself. The two main environments are the browser and Node.js.

### 1. In the Browser 🌐

This is the original and most common environment for JavaScript. Every modern web browser comes with a built-in JavaScript engine.

- **JS Engine Examples:** Google's **V8** (in Chrome, Edge), Mozilla's **SpiderMonkey** (in Firefox), Apple's **JavaScriptCore** (in Safari).
    
- **Purpose:** To make web pages interactive. JS code is typically embedded in an HTML file using a `<script>` tag or linked as an external `.js` file.
    
- **Key APIs:** Provides browser-specific APIs, collectively known as **Web APIs**. The most important are:
    
    >- `document` (DOM API): To manipulate the HTML and CSS of the page.
>        
>      - `window`: The global object that represents the browser window.
  >      
  >    - `fetch()`: To make network requests to servers.
  >      
 >    - `localStorage` / `sessionStorage`: To store data in the browser.
        

### 2. With Node.js 🖥️

Node.js is a runtime environment that allows you to ==run JavaScript **outside of the browser**, on a server or your personal computer==.

- **JS Engine:** Uses Google's V8 engine, the same one found in Chrome.
    
- **Purpose:** <mark style="background: #ABF7F7A6;">To build backend applications, APIs, scripts, and command-line tools</mark>.
    
- **Key APIs:** Provides APIs for server-side operations. It does **not** have access to browser APIs like `document` or `window`. Instead, it offers modules for:
    
>    - `fs` (File System): To read from and write to files.
  >      
>    - `http`: To create web servers and handle HTTP requests.
  >      
>    - `path`: To work with file and directory paths.
>    
   > - `process`: To get information about the current running process.
        

# **Memory Palace**
---
## **1. Chosen Location / Room**

**Kitchen**

## **2. Loci (Memory Spots)**

- **Spot 1:** The Stove Knobs (The Engine)
    
- **Spot 2:** The Left Burner (Browser / Web APIs)
    
- **Spot 3:** The Right Burner (Node.js / File System)
    

## **3. Encoded Imagery / Story**

- **Spot 1 (The Engine):** The knobs are tiny **V8 Engines**. I have to "crank" them like an old car to turn the stove on.
    
- **Spot 2 (Browser):** On the left burner, a **Spider** (SpiderMonkey) is spinning a web (Web APIs) to catch a **Window** that is floating away.
    
- **Spot 3 (Node.js):** On the right burner, there is no window, just a heavy **Green Safe** (File System). I’m trying to cook a "Node" (a knot of rope) inside the safe.
    

## **4. Retrieval Path**

Enter Kitchen → Reach for the Knobs (Engine) → Look at Left Burner (Browser) → Look at Right Burner (Node).

# Related Concepts:
---

- **JavaScript Engine:** The core component that executes JS code (e.g., V8). It's the "motor". The runtime environment is the "car" – it includes the engine plus other necessary parts (APIs).
    
- **Runtime Environment:** The complete system where JS code runs. It bundles the JS Engine with external APIs (Web APIs in browsers, Node.js APIs on the server).
    
- **Client-Side vs. Server-Side:** This dichotomy is defined by the runtime. **Client-side** code runs on the user's machine (the browser). **Server-side** code runs on a remote server (Node.js).
    
- **[[JavaScript What is the DOM (Document Object Model)]]:** A tree-like representation of an HTML document. It's a key Web API provided by the browser runtime, allowing JavaScript to interact with and change the content of a webpage. It is unavailable in Node.js.
    
- **npm (Node Package Manager):** The default package manager for Node.js. It's a command-line tool and a public registry of reusable JavaScript code packages, forming the core of the Node.js ecosystem.
    

# Examples:
---
### Example 1: Running JavaScript in a Browser (inside HTML)

```HTML
<!DOCTYPE html>
<html lang="en">
<head>
    <title>Browser JS Example</title>
</head>
<body>
    <h1 id="greeting">Hello, World!</h1>
    <button id="myButton">Click Me</button>

    <script>
        // Access the button and heading elements using the 'document' object (a Web API)
        const myButton = document.getElementById('myButton');
        const greeting = document.getElementById('greeting');

        // Add an event listener to the button
        // When the button is clicked, this function will run
        myButton.addEventListener('click', () => {
            // Change the text content of the h1 element
            // This is an example of DOM manipulation
            greeting.textContent = 'Hello, JavaScript!';
        });
    </script>
</body>
</html>
```

### Example 2: Running JavaScript with Node.js

```JavaScript
// To run this file, save it as `app.js` and run `node app.js` in your terminal.

// 1. Import the built-in 'fs' (File System) module from Node.js.
//    This module is a Node.js API and is NOT available in the browser.
const fs = require('fs');

// 2. Define a file path and some content to write.
const filePath = 'example.txt';
const fileContent = 'Hello from Node.js! This was written by a script.';

// 3. Use the writeFile method from the fs module to create a new file and write content to it.
//    This is an asynchronous operation. It takes the file path, content, and a callback function.
fs.writeFile(filePath, fileContent, (err) => {
    // 4. This callback function runs after the file-writing operation is complete.
    //    It checks if an error occurred.
    if (err) {
        console.error('An error occurred:', err);
        return;
    }

    // 5. If successful, log a message to the console.
    //    'console.log' works in both Node.js and browsers, but here it outputs to the terminal.
    console.log(`File "${filePath}" has been saved successfully!`);
});

// This message will likely appear first, because writeFile is asynchronous.
console.log('Attempting to write a file...');
```

# Flashcards:
---

What are the two primary runtime environments for JavaScript?;;
In the Browser (client-side for web interactivity) and Node.js (server-side for backend applications).

What is a JavaScript engine?;; A program that executes JavaScript code. Examples include Google's V8 (used in Chrome and Node.js) and Mozilla's SpiderMonkey (used in Firefox).

What global object, representing the webpage, is available in a browser environment but not in Node.js?;; The document object, which provides the DOM (Document Object Model) API for manipulating HTML elements.

What fundamental capability does Node.js have that browser JavaScript does not?;; Direct access to the server's file system (e.g., using the fs module), operating system, and other low-level resources.

How do you run a JavaScript file named app.js using Node.js in the terminal?;; By typing the command: node app.js.

What is the main difference between a JS engine and a JS runtime environment?;; The engine (e.g., V8) just executes the code. The runtime environment (e.g., Browser, Node.js) provides the engine plus specific APIs to interact with the outside world (like the DOM API in browsers or the fs API in Node.js).