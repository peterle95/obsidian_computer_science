---
memory: to_finish
tags:
  - learning
language:
  - JavaScript
review-date: 2025-11-25
last-reviewed: 2025-10-15
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

# Core Explanation:
---

An **event** is a signal sent by the browser that something has happened. These signals can be triggered by user interactions (e.g., clicking a button) or by the browser itself (e.g., the page finishing loading).

To make our code react to these signals, we use an **event listener**. This is a JavaScript function that "listens" for a specific event to happen on a specific HTML element. The modern and standard way to attach a listener is with the `element.addEventListener()` method.

This method takes two primary arguments:

1. **The event type (a string):** The name of the event to listen for (e.g., `'click'`).
    
2. **The listener function (a callback):** The function to execute when the event occurs.
    

### Common Event Types

- ==**`click`**==: Fired when a user presses and releases the primary mouse button on an element. It's the most common event for buttons, links, and other interactive elements.
    
- ==**`mouseover`**==: Fired when the user moves the mouse cursor onto an element. Often used for hover effects or tooltips.
    
- ==**`keydown`**==: Fired when a <mark style="background: #D2B3FFA6;">user presses any key on the keyboard</mark>. Useful for capturing user input in real-time or creating keyboard shortcuts.
    
- ==**`submit`**==: Fired on a `<form>` element <mark style="background: #D2B3FFA6;">when it's submitted</mark> (either by clicking a `type="submit"` button or pressing Enter in a text field). This allows you to intercept the submission to perform validation or send data with AJAX.
    
- ==**`load`**==: Fired on the `window` object when the entire page, including all dependent resources like stylesheets and images, has completely loaded. This is crucial for ensuring code that manipulates elements only runs after those elements exist.

- ==**change**==:  Fired when the <mark style="background: #ABF7F7A6;">value of an input element changes and the element loses focus.</mark> This event is triggered <mark style="background: #ABF7F7A6;">after the user modifies the content of form elements like text inputs, textareas, select dropdowns, or checkboxes, and then clicks elsewhere or tabs away</mark>. Unlike the `input` event which fires immediately as the user types, `change` only fires once the user is done editing and moves focus away from the element. <mark style="background: #ABF7F7A6;">This makes it ideal for validation, saving data, or triggering actions that should only occur after the user has finished making their changes.</mark>

# Related Concepts:
---

- **Event Handler/Listener**: This is the callback function that is registered to run when an event is fired. It's the "what to do" part of event handling.
    
- **Event Object**: When an event occurs, the browser automatically creates an `event` object and passes it as the first argument to the listener function. This object contains valuable information about the event, such as the target element (`event.target`), the key pressed (`event.key`), or mouse coordinates.
    
- **Event Bubbling and Capturing**: The process that determines the order in which event listeners on nested elements are fired. By default, events "bubble" up from the innermost element to the outermost.
    
- **`event.preventDefault()`**: ==A method on the event object that stops the browser's default action for that event. It's most commonly used with the `submit` event to prevent a form from reloading the page.==
    
- **`innerText`**: This is a **property** of a DOM element, **not an event type**. It is frequently used _inside_ event handlers to get or set the visible text content of an element. For example, on a `click` event, you might change the `innerText` of a `<p>` tag.
    

# Examples:
---

```HTML
<!DOCTYPE html>
<html lang="en">
<head>
    <title>JS Events Demo</title>
    <style>
        /* Simple styling for the mouseover example */
        #hover-box { width: 200px; height: 100px; background-color: lightblue; text-align: center; line-height: 100px; }
        .highlight { background-color: gold; }
    </style>
</head>
<body>

    <h1>JavaScript Event Examples</h1>

    <button id="click-button">Click Me</button>
    <p id="click-output"></p>

    <div id="hover-box">Hover Over Me</div>

    <input type="text" id="key-input" placeholder="Type here...">
    <p id="key-output"></p>
    
    <form id="my-form">
        <input type="text" id="name-input" placeholder="Enter name and submit">
        <button type="submit">Submit</button>
    </form>
    <p id="form-output"></p>

    <script>
        // We wait for the 'load' event on the window to ensure all HTML elements are available before we try to add listeners to them.
        window.addEventListener('load', () => {
            console.log('Page has fully loaded. Running script.');

            // --- 1. CLICK Event ---
            const clickButton = document.getElementById('click-button');
            const clickOutput = document.getElementById('click-output');

            // Add a listener that waits for a 'click' on the button.
            clickButton.addEventListener('click', () => {
                // When clicked, update the innerText of the paragraph.
                clickOutput.innerText = 'Button was clicked at ' + new Date().toLocaleTimeString();
            });

            // --- 2. MOUSEOVER Event ---
            const hoverBox = document.getElementById('hover-box');

            // Add a listener for when the mouse enters the div's area.
            hoverBox.addEventListener('mouseover', () => {
                // Change the text and add a CSS class for highlighting.
                hoverBox.innerText = 'Mouse is here!';
                hoverBox.classList.add('highlight');
            });
            
            // We can also listen for 'mouseout' to revert the changes.
            hoverBox.addEventListener('mouseout', () => {
                hoverBox.innerText = 'Hover Over Me';
                hoverBox.classList.remove('highlight');
            });

            // --- 3. KEYDOWN Event ---
            const keyInput = document.getElementById('key-input');
            const keyOutput = document.getElementById('key-output');
            
            // Listen for any key being pressed down inside the input field.
            // The listener function receives the 'event' object automatically.
            keyInput.addEventListener('keydown', (event) => {
                // The event object has a 'key' property that tells us which key was pressed.
                keyOutput.innerText = `You just pressed: "${event.key}"`;
            });

            // --- 4. SUBMIT Event ---
            const myForm = document.getElementById('my-form');
            const nameInput = document.getElementById('name-input');
            const formOutput = document.getElementById('form-output');
            
            // Listen for the 'submit' event on the form itself, not the button.
            myForm.addEventListener('submit', (event) => {
                // VERY IMPORTANT: Prevent the browser's default form submission behavior (which reloads the page).
                event.preventDefault();

                // Now we can handle the form data with JavaScript.
                const submittedName = nameInput.value;
                formOutput.innerText = `Hello, ${submittedName}! The form was submitted without a page refresh.`;
            });
        });
    </script>
</body>
</html>
```

# Flashcards:
---

What is a JavaScript event?;; A signal that something has happened, either a user action (like a 'click') or a browser action (like a 'load'), which JS can detect and react to.

What method is used to make an element "listen" for an event and run a function in response?;; element.addEventListener('eventType', functionToRun);

What is the difference between the click and mouseover events?;; click fires when a user presses and releases the mouse button on an element. mouseover fires when the mouse pointer simply moves onto the element's area.

You need to stop a form from reloading the page when submitted. What event do you listen for, and what method do you call on the event object?;; Listen for the submit event on the <form> element and call event.preventDefault() inside the handler function.

You want to run a script only after the entire webpage, including all images, has fully loaded. Which event should you listen for, and on what object?;; The load event on the global window object (e.g., window.addEventListener('load', ...)).

Is innerText an event type?;; No, it is a property of an HTML element. It is often used inside an event handler function to get or set the visible text content of that element.