---
memory: to_finish
tags:
  - to_learn
language:
  - React
review-date:
last-reviewed:
keywords:
  - components
  - composition
  - reusable UI
  - component tree
  - JSX
scheda: to_finish
palace: Finowstraße
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
Components solve the problem of building large user interfaces without turning the code into one huge, tangled page.

A modern UI is made of many repeated or related pieces: buttons, cards, navbars, forms, sidebars, modals, list items, profile previews, and full pages. Without components, you would have to write and maintain all of that markup, styling, and behavior as one large structure. That becomes hard to read, hard to reuse, and easy to break.

React's component model lets you break the UI into small, named, reusable pieces. Each component describes one part of the interface, and larger screens are created by composing smaller components together.

This matters because components make React apps:

- easier to understand, because each part has a clear name and purpose;
- easier to reuse, because the same component can appear in many places;
- easier to test and debug, because problems are isolated to smaller pieces;
- easier to scale, because large pages become trees of smaller UI units;
- easier to change, because updating one component updates every place that uses it.

A component is not only a visual block. It is a unit of UI logic: it can receive data, decide what to display, render markup, and later connect to state, events, effects, or routing.

# **Core Explanation:**
---
A React component is a reusable piece of user interface. In modern React, components are usually JavaScript functions that return JSX.

At a high level, a component answers this question:

> Given some data, what should this part of the UI look like?

Important ideas:

1. **Components are UI building blocks**
   A component can be tiny, like a button, or large, like an entire page. Small components can be combined into bigger components.

2. **Components are reusable**
   You define a component once and use it many times. For example, one `UserCard` component can render many different users if it receives different data.

3. **Components are composable**
   Components can render other components. A `Dashboard` can render `Sidebar`, `StatsPanel`, `ActivityFeed`, and `UserMenu` components.

4. **Components form a tree**
   React UIs are structured as a component tree. A parent component renders child components. Those child components can render their own children.

5. **Component names must be capitalized**
   In JSX, lowercase tags such as `<section>` or `<button>` mean built-in HTML elements. Capitalized tags such as `<ProfileCard />` or `<Navbar />` mean custom React components.

6. **Components should be declared at the top level**
   Do not define one component inside another component. Define components at the top level of the file, then compose them by rendering them inside each other. This avoids confusing bugs and unnecessary recreation of component definitions.

7. **Components can be organized into files**
   Small related components can sometimes live in the same file. As a project grows, important reusable components should be moved into their own files and imported where needed.

8. **Components are not the same thing as props or state**
   A component is the UI unit. [[React - Props (Properties)]] are data passed into the component. [[React - State]] is data the component manages over time. Components become powerful because they can combine structure, props, state, events, and rendering.

This note is the broad concept note for components. Details about writing modern function components belong in [[React - Functional Components]]. Details about older class syntax belong in [[React - Class Components (Legacy)]].

# **Memory Palace**
---
## **1. Chosen Location / Room**
_Choose the palace room later._

## **2. Loci (Memory Spots)**
_List 1-3 physical spots in that location that you will attach information to._

- Spot 1:
- Spot 2:
- Spot 3:

## **3. Encoded Imagery / Story (Visual OR Non-Visual)**
_Describe the mnemonic you attach to each spot after choosing the loci._

- Spot 1 mnemonic:
- Spot 2 mnemonic:
- Spot 3 mnemonic:

## **4. Retrieval Path**
_Write a clear retrieval route after choosing the loci._

# **Related Concepts:**
---
- **[[React - Thinking in Components]]**: The mental habit of looking at a UI and breaking it into independent pieces before building it.

- **[[React - Functional Components]]**: The modern way to define React components using JavaScript functions. This should contain the deeper syntax and best practices for function components.

- **[[React - Class Components (Legacy)]]**: The older way to define components using JavaScript classes. Class components are still supported but are not recommended for new React code.

- **[[React - JSX (JavaScript XML)]]**: The syntax components usually return. JSX lets a component describe markup inside JavaScript.

- **[[React - Props (Properties)]]**: The mechanism for passing data from a parent component to a child component.

- **[[React - State]]**: Data that belongs to a component and can change over time, causing the UI to update.

- **[[React - The Virtual DOM (VDOM)]]**: The in-memory representation React uses when deciding how to update the real DOM after components render.

# **Examples:**
---
## **Example 1: A small reusable component**
```jsx
// This is a React component because it is a capitalized function
// that returns JSX describing part of the UI.
function WelcomeMessage() {
  // The returned JSX describes what this component should display.
  return <h1>Welcome to the app!</h1>;
}

// This larger component renders the smaller WelcomeMessage component.
// The capitalized <WelcomeMessage /> tag tells React this is a custom component.
export default function App() {
  return (
    <main>
      <WelcomeMessage />
    </main>
  );
}
```

## **Example 2: Composing several components into a page**
```jsx
// Header is responsible only for the top area of the page.
function Header() {
  return <header>My Dashboard</header>;
}

// Sidebar is responsible only for navigation.
function Sidebar() {
  return <aside>Navigation links go here</aside>;
}

// Content is responsible only for the main page content.
function Content() {
  return <section>Main dashboard content goes here</section>;
}

// Dashboard composes the smaller components into one screen.
// This keeps each part easier to understand and reuse.
export default function Dashboard() {
  return (
    <div>
      <Header />
      <Sidebar />
      <Content />
    </div>
  );
}
```

## **Example 3: Capitalization matters in JSX**
```jsx
// This is a custom React component because the name starts with a capital letter.
function ProfileCard() {
  return <article>Profile information</article>;
}

export default function App() {
  return (
    <main>
      {/* Lowercase <article> means a built-in HTML element. */}
      <article>This is a normal HTML article element.</article>

      {/* Capitalized <ProfileCard /> means a custom React component. */}
      <ProfileCard />
    </main>
  );
}
```

## **Example 4: Do not define a component inside another component**
```jsx
// Good: both components are defined at the top level of the file.
function UserAvatar() {
  return <img src="/avatar.png" alt="User avatar" />;
}

export default function UserProfile() {
  // UserProfile can render UserAvatar without defining it inside the function.
  // This keeps component identity stable and avoids confusing bugs.
  return (
    <section>
      <UserAvatar />
      <p>Username</p>
    </section>
  );
}
```

# **Flashcards:**
---

What problem do React components solve?;;They let you break a large UI into smaller, named, reusable pieces that are easier to understand, reuse, debug, and maintain.

What is a React component?;;A reusable piece of user interface, usually written as a JavaScript function that returns JSX.

What does it mean that components are composable?;;Components can render other components, allowing small UI pieces to be combined into larger screens and full applications.
Why must custom React component names start with a capital letter?;;In JSX, lowercase tags are treated as built-in HTML elements, while capitalized tags are treated as custom React components.
Why should components be declared at the top level instead of inside other components?;;Top-level declarations keep component identity stable, avoid unnecessary recreation, and prevent confusing bugs.
How are components different from props and state?;;A component is the UI unit; props are data passed into it, and state is data it manages over time.