---
memory: to_finish
tags:
  - learning
language:
  - React
review-date: 2025-11-25
last-reviewed: 2025-10-16
scheda: done
visit-count: 3
confidence-level: 1
consecutive-correct: 0
last-struggle-date: 2025-10-16
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
"Thinking in Components" is the core ==**design philosophy** of React==. It's a mental model for ==breaking down a complex user interface into a tree of simple, reusable, and isolated pieces.==

The fundamental problem it ==solves is the chaos of building and maintaining large, monolithic UI codebases==. Without this approach, a UI becomes a tangled mess of interconnected parts, making it difficult to update, debug, or reuse code. A small change in one area can unexpectedly break another.

This concept is important because it forces you to create a **well-structured, modular, and scalable application from the start**. ==It shifts your perspective from "what does the whole page look like?" to "what are the individual, logical building blocks of this page?"==. This leads to cleaner, more maintainable code and promotes reusability (e.g., the same `Button` component can be used in dozens of different places).

# **Core Explanation:**

---
"Thinking in Components" is a repeatable, five-step process for translating a UI design mockup into a component hierarchy.

1. **Step 1: ==Break The UI Into a Component Hierarchy==** Start with your UI design. <mark style="background: #BBFABBA6;">Draw boxes around every logical piece of the UI and give it a name. If a component inside a box seems too complex, break it down further into smaller sub-components. You are creating a visual tree of your application's structure.</mark>

2. **Step 2: ==Apply the Single Responsibility Principle (SRP)==** Look at your boxes. <mark style="background: #BBFABBA6;">A component should ideally do only one thing</mark>. If a component is responsible for displaying data _and_ handling user input _and_ filtering a list, it’s a sign you should split it into smaller components (e.g., `DisplayList`, `FilterInput`).

3. **Step 3: ==Create a Static Version in React==** Build a version of your application that renders the UI with your defined component hierarchy but has no interactivity. Pass data using **props**, flowing from parent components down to child components. Don't use **state** yet. This step is about getting the structure right.

4. **Step 4: ==Identify The Minimal (but complete) Representation Of UI [[React - State]]==** Now, <mark style="background: #BBFABBA6;">think about the data that can change over time and is not passed down from a parent. This is your application's **state**. Identify the absolute minimum set of stateful data your app needs to represent its UI. For example, in a search component, the text the user has typed is state, but the filtered list of results is not—it can be _computed_ from the state</mark>. Ask three questions for each piece of data:
 > - Is it passed in from a parent via props? (If so, it probably isn’t state.)
 > - Does it remain unchanged over time? (If so, it probably isn’t state.)
 > - Can you compute it based on other state or props? (If so, it isn’t state.)
5. **Step 5: ==Identify Where Your State Should Live==** For each piece of state, identify which component "owns" it. The state should live in the common ancestor component of all components that need that data. Data flows down from the state-owning component to child components via props. Actions that modify the state (like `onClick` handlers) are passed down as prop functions from the owner to the components that need to trigger changes.

# **Related Concepts:**

---
- **Component-Based Architecture:** The broader software design paradigm where an application is built from loosely coupled, independent components. "Thinking in Components" is the practical application of this paradigm in React.

- **Single Responsibility Principle (SRP):** A fundamental principle of software design stating that every module or class should have responsibility over a single part of the functionality. In React, this is the primary guideline for deciding how to draw your component boundaries.

- **State vs. Props:** This concept is central to "Thinking in Components".

 - **[[React - Props (Properties)]]** (`properties`): Data passed _down_ from a parent component to a child. The child component cannot change its props.
 - **State:** Data managed _within_ a component that can change over time (e.g., user input, a fetched API response). Identifying what is state and where it lives (Steps 4 & 5) is a critical part of the process.
- **Composition:** The act of combining simple components to create more complex components. "Thinking in Components" naturally leads to a compositional model, which is more flexible than classical inheritance.

# **Examples:**

---

Our component hierarchy would be:

- `Video` (The main container)
 - `VideoPlayer` (Shows the video itself, takes a video URL)
 - `VideoTitle` (Shows the title, takes a string)
 - `VideoInfo` (Shows view count, takes a number)
 - `ActionBar` (A container for buttons)
 - `ActionButton` (A reusable button for Like, Dislike, Comment)
```js
import React from 'react';

// This is a reusable, low-level component. It's so simple,
// it might just be a styled button in a real app.
// It receives its label and count via props.
function ActionButton({ label, count }) {
 return <button>{label} {count && `(${count})`}</button>;
}

// The ActionBar's single responsibility is to arrange the action buttons.
// It owns no state itself; it just renders the buttons it's told to.
function ActionBar({ commentCount }) {
 return (
 <div>
 <ActionButton label="👍 Like" />
 <ActionButton label="👎 Dislike" />
 <ActionButton label="💬 Comment" count={commentCount} />
 </div>
 );
}

// The main "smart" component that gathers all the data
// and composes the smaller, "dumb" components together.
// It owns the data and passes it down as props.
function Video({ videoData }) {
 return (
 // We use a React Fragment <> to group elements without adding an extra div to the DOM.
 <>
 {/* The VideoPlayer would likely be a more complex component */}
 <div>Video Player for {videoData.url}</div>

 {/* Each sub-component receives only the props it needs. */}
 <h2>{videoData.title}</h2>
 <p>{videoData.views} views</p>

 {/* The Video component passes the comment count down to the ActionBar. */}
 <ActionBar commentCount={videoData.comments} />
 </>
 );
}

// The final App component would render the Video, passing in the initial data.
// In a real app, this data would come from an API.
export default function App {
 const videoData = {
 url: '.com/video123',
 title: 'The Ultimate Guide to React',
 views: 150000,
 comments: 32,
 };

 return <Video videoData={videoData} />;
}
```

# **Flashcards:**

---
What is the very first step in "Thinking in Components"?;; Start with a UI design or mockup and draw boxes around every logical component and sub-component to define your component hierarchy.

What is the main principle used to decide if a component should be broken down further?;; The Single Responsibility Principle (SRP): a component should ideally only do one thing.

Where should a piece of state live in a React component tree?;; It should live in the lowest common ancestor of all the components that need to read or update that state.