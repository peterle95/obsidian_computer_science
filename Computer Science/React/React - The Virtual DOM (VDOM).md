---
memory: to_finish
tags:
  - learned
language:
  - React
review-date:
last-reviewed: 2025-10-24
scheda: done
visit-count: 3
confidence-level: 2.5
consecutive-correct: 3
last-struggle-date: ""
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

# **Purpose/Why**:
---

The Virtual DOM (VDOM) is a programming concept implemented in frameworks like React to <mark style="background: #BBFABBA6;">solve the performance bottlenecks associated with direct manipulation of the real Document Object Model (DOM)</mark>. <mark style="background: #BBFABBA6;">Manipulating the real DOM is slow because any change can trigger a cascade of updates, including re-calculating CSS, layout changes (reflow), and repainting the screen.</mark> For complex applications with frequent UI updates, these direct manipulations can lead to a sluggish user experience.

<mark style="background: #BBFABBA6;">The VDOM's primary application is to minimize these expensive real DOM operations. </mark>It acts as a lightweight, in-memory representation of the real DOM. <mark style="background: #BBFABBA6;">When the state of a component changes, React first updates this virtual representation, which is much faster than updating the actual DOM. It then compares the new VDOM with a snapshot of the previous one to identify the minimal set of changes required. Only these specific changes are then applied to the real DOM in a batched and optimized manner. This process, known as reconciliation, makes the UI update process significantly more efficient and improves application performance.</mark>

# **Core Explanation:**
---

The Virtual DOM (VDOM) is a lightweight, in-memory copy of the real DOM. Instead of being tied to the browser's rendering engine, it's a JavaScript object that mirrors the structure of the real DOM.

**Key Characteristics:**

- **Lightweight:** It's a plain JavaScript object, making it much faster to create and manipulate than the real DOM.

- **In-Memory:** The VDOM resides in memory and is not directly rendered on the screen.

- **Declarative:** Developers declare what the UI should look like for a given state, and React takes care of updating the actual DOM to match that state, abstracting away the imperative DOM manipulation.


**How it Works:**

The process of updating the UI using the VDOM can be broken down into three main steps:

1. **State Change and VDOM Creation:** When the state of a component changes (e.g., through a setState call), React creates a new VDOM tree representing the updated UI.

2. **Diffing:** React then compares this new VDOM tree with a snapshot of the previous VDOM tree. This comparison process is called "diffing." React uses an efficient diffing algorithm to find the minimal number of changes between the two trees.

3. **Reconciliation:** Based on the differences identified during the diffing process, React determines the most efficient way to update the real DOM. It then batches these updates and applies them to the real DOM in a single operation. This reconciliation process ensures that only the necessary parts of the actual DOM are changed.

# **Related Concepts:**
---

- **Real DOM:** This is the actual tree-like structure of HTML elements that the browser uses to render the user interface. The VDOM is a lightweight copy of the Real DOM and is used to optimize updates to it. Direct manipulation of the Real DOM is what the VDOM aims to minimize.

- **Reconciliation:** This is the process React uses to update the real DOM to match the most recent state of the component. It involves creating a new VDOM, comparing it with the old one (diffing), and then applying the minimal necessary changes to the real DOM.

- **Diffing Algorithm:** This is the specific algorithm React uses during the reconciliation process to compare two VDOM trees and identify what has changed. It relies on heuristics to achieve a highly efficient comparison, such as assuming that two elements of different types will produce different trees.

- **Shadow DOM:** This is a browser technology designed for encapsulation in web components. It allows a component to have its own separate, "shadow" DOM tree that is isolated from the main document's DOM. While both the VDOM and Shadow DOM deal with the DOM, their purposes are different. The VDOM is a performance optimization for UI updates, while the Shadow DOM is for scoping and component encapsulation.

# **Examples:**
---

```js
import React, { useState } from 'react';
import ReactDOM from 'react-dom';

// Simple component to demonstrate the Virtual DOM in action.
function Counter() {
  // 'count' is our state. When it changes, React will trigger a re-render.
  const [count, setCount] = useState(0);

  // When the button is clicked, we update the state.
  const handleIncrement = () => {
    setCount(count + 1);
  };

  // JSX is used to describe what the UI should look like.
  // This JSX gets converted into React elements, which form the Virtual DOM.
  return (
    <div>
      <h1>Counter: {count}</h1>
      <button onClick={handleIncrement}>Increment</button>
      <p>
        When you click the "Increment" button, the `count` state updates.
        React then creates a new Virtual DOM tree with the updated count.
        It compares this new VDOM with the previous one and sees that only the text inside the `<h1>` has changed.
        Finally, React updates only that specific text in the real DOM, leaving the rest of the page untouched.
        This is much more efficient than re-rendering the entire component in the real DOM.
      </p>
    </div>
  );
}

// Render the component to the real DOM.
ReactDOM.render(<Counter />, document.getElementById('root'));
```

# **Flashcards:**

---
What is the primary problem the Virtual DOM solves?;;The Virtual DOM solves the performance issues associated with direct and frequent manipulation of the real DOM, which can be slow and lead to a poor user experience.

What is the Virtual DOM?;;The Virtual DOM (VDOM) is a lightweight, in-memory representation of the real DOM, existing as a JavaScript object.

Briefly describe the three main steps of how the Virtual DOM works.;;1. A new VDOM is created when state changes. 2. This new VDOM is compared (diffed) with the previous VDOM. 3. The minimal necessary changes are then applied to the real DOM in a process called reconciliation.

What is the difference between the Virtual DOM and the Real DOM?;;The Real DOM is the actual HTML structure the browser renders, while the Virtual DOM is a lightweight, in-memory copy used by React to optimize updates to the Real DOM.

What is "reconciliation" in React?;;Reconciliation is the process React uses to efficiently update the real DOM. It involves comparing the old Virtual DOM with the new Virtual DOM and then applying only the necessary changes to the actual DOM.

Is the Virtual DOM the same as the Shadow DOM?;;No. The Virtual DOM is a performance optimization concept for UI updates, whereas the Shadow DOM is a browser technology for encapsulating the styles and structure of web components.
