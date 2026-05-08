---
memory: done
tags:
  - learned
language:
  - React
review-date:
last-reviewed: 2025-08-27
scheda: done
visit-count: 4
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-08-05
palace: Finowstraße
palace-room: Entrance
locus: Entrance door
palace-order: 100
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
The fundamental problem React solves is the **complexity of building and maintaining dynamic user interfaces (UIs)**, especially in large-scale applications.

Before libraries like React, ==web developers manually manipulated the **[[JavaScript What is the DOM (Document Object Model)]]**. This is an **<mark style="background:

FF5582A6;">imperative</mark>** approach: you write step-by-step instructions to find an element, create a new one, change its style, add an event listener,== etc. As applications grew (e.g., social media feeds, interactive dashboards), this manual process became incredibly complex, slow, and prone to bugs. Keeping the UI consistent with the underlying application data was a significant challenge.

React's importance lies in introducing a **<mark style="background:

FF5582A6;">declarative, component-based</mark>** model:

1. **Declarative:** <mark style="background:

BBFABBA6;">You simply describe</mark> _what_ <mark style="background:

BBFABBA6;">the UI should look like for any given state of your application. You don't tell React</mark> _how_ <mark style="background:

BBFABBA6;">to update the UI. When the data changes, React automatically and efficiently updates the necessary parts of the DOM to match the new state</mark>.
2. **Component-Based:** It allows you to <mark style="background:

D2B3FFA6;">break down a complex UI into small, isolated, and reusable pieces called "components."</mark> This makes your code more organized, easier to debug, and scalable.

In essence, React's purpose is to let developers build complex UIs by composing simple, stateful components, abstracting away the difficult and error-prone parts of direct DOM manipulation.

# **Core Explanation:**

---
**React** is a free and open-source JavaScript library for building user interfaces. It is not a full framework, meaning it focuses specifically on the "view" layer of an application.

**Key Characteristics:**

1. **Component-Based Architecture:** The core principle of React is building UIs out of components. A component is a self-contained, reusable piece of code that manages its own state and renders a part of the UI. For example, a chat application's UI could be broken down into components like `Sidebar`, `ContactList`, `ChatMessage`, and `MessageInput`.

2. **Declarative Syntax (using JSX):** React uses [[React - JSX (JavaScript XML)]], a syntax extension that allows you to write HTML-like code directly within your JavaScript. You declare the UI's structure based on the current data (state and props). When that data changes, React re-renders the component with the new data.

3. **[[React - The Virtual DOM (VDOM)]]:** ==Direct DOM manipulation is slow. To solve this, React keeps a lightweight representation of the UI in memory, called the Virtual DOM. When a component's state changes, React first creates a new Virtual DOM tree. It then compares this new VDOM with the previous one (a process called "diffing") to figure out the most efficient, minimal set of changes needed for the real DOM. Finally, it applies only these changes, which is significantly faster than re-rendering the entire UI.==

# **Related Concepts:**

---
- **DOM (Document Object Model):** The tree-like structure created by the browser to represent a web page's HTML. React intelligently interacts with the DOM on your behalf so you don't have to do it manually.

- **[[React - What is a Single Page Application (SPA)]]:** An application that loads a single HTML page and dynamically updates its content as the user interacts with it, without full page reloads. React is the leading library for building SPAs because its component-based model and virtual DOM are perfect for efficiently managing these dynamic updates.

- **JSX (JavaScript XML):** The syntax used to write React components. It looks like HTML but is actually transformed by a tool like Babel into regular JavaScript (`React.createElement` calls). It's the mechanism for _declaring_ the UI in React.

- **Imperative vs. Declarative Programming:**

 - **Imperative (The "How"):** Manually manipulating the DOM is imperative. _Example: "Find the element with ID 'title', create a text node 'Hello', and append it."_
 - **Declarative (The "What"):** React is declarative. _Example: "If the `isLoggedIn` state is true, the heading should be 'Welcome back!'"_ You declare the desired outcome, and React handles the "how."

# **Memory Palace**
---
## **1. Encoded Imagery / Story (Visual OR Non-Visual)**

You walk up to the entrance door and try to open it, but the door is covered in tiny sticky notes that say things like: "change title," "hide button," "add new div," "update color," and "please do not forget the click listener." Every time you peel one off, three more appear. This is the old manual DOM way: too many small instructions, too easy to mess up.

Then the door suddenly puts on a little project-manager voice and says: "Stop telling me every tiny step. Just tell me what the entrance should look like." You say: "If the user is logged in, show Welcome. If not, show Login." The door instantly rearranges itself without you touching the sticky notes. This is React's declarative idea: you describe the desired UI, and React handles the updates.

Finally, the door splits into three neat reusable panels: a handle panel, a nameplate panel, and a lock panel. Each panel can be removed, reused, and replaced somewhere else in the building. That is the component idea: complex UI becomes small reusable pieces. Behind the door, a faint shadow-door copies every movement first before the real door changes. That shadow-door is the Virtual DOM, helping React figure out the smallest real change to make.

- Spot 1 mnemonic: The overloaded door handle covered in sticky notes represents painful manual DOM manipulation.
- Spot 2 mnemonic: The talking door that asks only what it should look like represents React's declarative model.
- Spot 3 mnemonic: The door opening into reusable panels and a shadow-door represents components plus the Virtual DOM.

## **2. Retrieval Path**
Approach the entrance door -> grab the sticky-note-covered handle -> listen to the talking declarative door -> open it into reusable panels -> notice the shadow-door behind it.


# **Examples:**
---

## **Example 1: The "Old Way" (Imperative Vanilla JavaScript)**
```html
<div id="root"></div>

<script>
 // 1. Get a reference to the DOM element where we want to place content.
 const rootElement = document.getElementById('root');

 // 2. Create a new h1 element programmatically.
 const heading = document.createElement('h1');

 // 3. Set its text content.
 heading.textContent = 'Hello, World!';

 // 4. Give it a class for styling.
 heading.className = 'greeting';

 // 5. Append the newly created element to our root container.
 // This is a direct, step-by-step manipulation of the DOM.
 rootElement.appendChild(heading);
</script>
```

## **Example 2: The "React Way" (Declarative with JSX)**
```js
// This code assumes you have a React environment set up.
import React from 'react';
import ReactDOM from 'react-dom/client';

// 1. We define a "Component" - a reusable piece of UI.
// This function RETURNS a description of what the UI should look like.
// It doesn't perform any step-by-step DOM manipulation itself.
function Greeting {
 // 2. We use JSX to declaratively write what we want to render.
 // This looks like HTML, but it's JavaScript.
 return <h1 className="greeting">Hello, React!</h1>;
}

// 3. Get a reference to the root DOM element (this is usually done only once in an app).
const rootElement = document.getElementById('root');

// 4. Create a React root.
const root = ReactDOM.createRoot(rootElement);

// 5. Tell React to render our <Greeting /> component inside the root.
// React takes our declarative component and handles the imperative DOM
// manipulation (createElement, appendChild, etc.) behind the scenes.
root.render(<Greeting />);
```

# **Flashcards:**

---
What fundamental problem does React solve?;; It solves the complexity of building and maintaining dynamic User Interfaces by replacing manual, error-prone DOM manipulation with a declarative, component-based model.

What does it mean for React to be "declarative"?;; You describe WHAT the UI should look like based on the current data (state), and React automatically handles the HOW of updating the DOM to match that state.

What is the role of the Virtual DOM (VDOM)?;; It's a lightweight copy of the UI kept in memory. React updates the VDOM first, calculates the most efficient changes needed for the real DOM, and then applies only those minimal changes, making updates very fast.