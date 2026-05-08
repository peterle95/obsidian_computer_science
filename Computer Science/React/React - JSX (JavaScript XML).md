---
memory: to_finish
tags:
  - learning
language:
  - React
review-date: 2025-11-25
last-reviewed: 2025-10-08
scheda: done
visit-count: 6
confidence-level: 3
consecutive-correct: 4
last-struggle-date: 2025-08-16
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
JSX (JavaScript XML) fundamentally solves the problem of defining user interface structures in a way that is both declarative and tightly integrated with JavaScript logic. ==Before JSX, building dynamic UIs in JavaScript often involved verbose methods like `document.createElement()` or using string-based templating engines that separated logic from presentation==. This separation could make complex UIs difficult to manage, understand, and debug, as the relationship between the UI structure and the data/logic driving it was not immediately clear.

JSX's primary application is in libraries like React (where it originated) and Preact, allowing developers to ==write HTML-like syntax directly within their JavaScript code==. This ==combines the declarative nature of HTML with the full programmatic power of JavaScript.== It's important in computer science and particularly in modern web development because it promotes a component-based architecture, where UI elements are encapsulated with their rendering logic and state. This paradigm leads to more maintainable, scalable, and understandable codebases, especially for complex single-page applications. By allowing UI to be defined inline with JavaScript, JSX makes it easier to visualize the structure of the UI components and how data flows through them.
# **Core Explanation:**
---
<mark style="background: #FF5582A6;">JSX is a syntax extension for JavaScript. It's not a new programming language; rather, it's a syntactic sugar that allows developers to write XML/HTML-like code directly within JavaScript. </mark>This code then gets transformed ("transpiled") into regular JavaScript function calls during the build process, typically by a tool like Babel.

Key characteristics and how it works:

- **Declarative UI:** <mark style="background: #BBFABBA6;">JSX allows you to describe what the UI should look like, rather than imperatively defining each element and its properties.</mark> This makes UI code much more readable and easier to reason about.
- **Embedded Expressions:** You can embed any valid JavaScript expression within JSX by enclosing it in curly braces `{}`. This allows you to dynamically render data, conditionally render elements, or loop through arrays to create lists of elements.
- **HTML-like but not HTML:** While it looks like HTML, JSX has some differences:
    >- `class` becomes `className` (due to `class` being a reserved keyword in JS).
    >- `for` becomes `htmlFor` (due to `for` being a reserved keyword in JS).
    >- CamelCase for HTML attributes (e.g., `onClick` instead of `onclick`).
    >- Self-closing tags _must_ be explicitly closed (e.g., `<img />` instead of `<img>`).
    >- Adjacent JSX elements must be wrapped in a single parent element (or a `Fragment` in React) because JSX expressions must resolve to a single root element.
- **Transpilation - [[JavaScript Transpilers (e.g., Babel) and Polyfills - Overview]]:**<mark style="background: #FFB86CA6;"> JSX is not understood directly by web browsers. Before the browser can execute the code, a transpiler (like Babel) converts the JSX syntax into standard JavaScript</mark>. For instance, `<h1 className="title">Hello</h1>` might be transpiled into `React.createElement('h1', { className: 'title' }, 'Hello')`.
- **Component-Based:** JSX is primarily used in conjunction with component-based UI libraries like React. It allows you to define custom components with their own state and props, and then use them in your JSX as if they were native HTML elements (e.g., `<MyComponent />`).
- **No Runtime Overhead:** Since JSX is transpiled to regular JavaScript, there's no runtime penalty or additional overhead in the browser. It's purely a compile-time convenience.

In essence, JSX bridges the gap between JavaScript logic and UI structure, making it highly intuitive to build dynamic and interactive user interfaces within a JavaScript environment.
# **Related Concepts:**
---
- **React (or other UI Libraries like Preact, SolidJS):** JSX is most famously associated with and heavily utilized by React. While JSX is a separate specification, it's almost always used within the context of these libraries to define their components' render output. It's the primary way developers write UI in React.
- **Babel:** This is the most common JavaScript transpiler used to convert JSX into standard JavaScript that browsers can understand. Babel takes the JSX code and transforms it into `React.createElement()` calls (or similar functions depending on the configured runtime). It's a build-time dependency for projects using JSX.
- **[[React - The Virtual DOM (VDOM)]]:** (Specific to React, related to how JSX is used) The Virtual DOM is an in-memory representation of the actual DOM. When JSX is rendered, React creates a Virtual DOM tree. When the component's state changes, React creates a new Virtual DOM tree, efficiently compares it with the previous one ("diffing"), and then updates only the necessary parts of the real DOM. JSX defines the structure that forms this Virtual DOM.
- **Component-Based Architecture:** JSX strongly encourages a component-based approach to UI development. Each JSX block often represents a self-contained UI component, encapsulating its own logic and presentation. This modularity improves code organization, reusability, and maintainability.
- **Templating Engines (e.g., Handlebars, EJS, Pug):** These are alternative ways to define UI structure, often using a specific syntax to embed dynamic data into HTML templates. They typically operate by generating HTML strings. JSX differs because it's JavaScript itself; you're writing JavaScript expressions and logic directly within the UI structure, rather than just inserting data into a string template. This allows for more powerful programmatic control over the UI.
- **HTML:** JSX has an HTML-like syntax, making it familiar to web developers. However, as noted, it has specific differences (e.g., `className`, self-closing tags) that distinguish it from pure HTML. It's designed to be a bridge, not a replacement, for HTML.
# **Examples:**
---
```js
// Note: This code is written assuming a React environment where JSX is transpiled.
// It will not run directly in a browser without a build step (e.g., using Create React App or Babel).

import React from 'react'; // In a React project, you typically import React

// --- Example 1: Basic JSX Element ---
function Greeting(props) {
    // JSX directly embedded in JavaScript return statement
    // Looks like HTML, but it's JavaScript
    return (
        // className instead of class because 'class' is a reserved keyword in JS
        <div className="greeting-container">
            {/* JavaScript expression embedded in curly braces */}
            <h1>Hello, {props.name || "Guest"}!</h1>
            <p>Welcome to our application.</p>
            {/* Self-closing tags must be explicitly closed with a slash */}
            <hr />
        </div>
    );
}

// --- Example 2: Embedding JavaScript Expressions ---
function ProductCard(props) {
    const { productName, price, isInStock } = props;

    return (
        <div className="product-card">
            <h2>{productName}</h2>
            <p>Price: ${price.toFixed(2)}</p>
            {/* Conditional rendering using a ternary operator within JSX */}
            {isInStock ? (
                <span style={{ color: 'green', fontWeight: 'bold' }}>In Stock</span>
            ) : (
                <span style={{ color: 'red', fontWeight: 'bold' }}>Out of Stock</span>
            )}
            {/* Event handler: onClick is camelCase, and its value is a JS function */}
            <button onClick={() => alert(`Added ${productName} to cart!`)}>
                Add to Cart
            </button>
        </div>
    );
}

// --- Example 3: Rendering Lists with map() ---
function ShoppingList(props) {
    const items = props.items || [];

    return (
        <div className="shopping-list">
            <h3>My Shopping List</h3>
            <ul>
                {/* Using JavaScript's map() method to render a list of items */}
                {/* Each item in the list needs a unique 'key' prop for React to efficiently update lists */}
                {items.map((item, index) => (
                    <li key={index}>
                        {item.name} - Quantity: {item.quantity}
                    </li>
                ))}
            </ul>
            {/* Conditional rendering of a message if the list is empty */}
            {items.length === 0 && <p>Your shopping list is empty!</p>}
        </div>
    );
}

// --- Example 4: Fragment for returning multiple elements without a wrapper div ---
function MyFragmentExample() {
    return (
        // React.Fragment or its shorthand <> allows you to group multiple elements
        // without adding an extra DOM node to the output.
        <>
            <p>This is the first paragraph.</p>
            <p>This is the second paragraph.</p>
            {/* It still counts as a single root element for JSX parsing */}
        </>
    );
}

// --- How JSX is used in a typical React App (Conceptual Main Component) ---
function App() {
    const products = [
        { productName: "Laptop", price: 1200.00, isInStock: true },
        { productName: "Mouse", price: 25.50, isInStock: false },
        { productName: "Keyboard", price: 75.00, isInStock: true }
    ];

    const shoppingItems = [
        { name: "Milk", quantity: 1 },
        { name: "Bread", quantity: 2 },
        { name: "Eggs", quantity: 12 }
    ];

    return (
        <div className="app-container">
            {/* Using our custom components defined with JSX */}
            <Greeting name="Alice" />
            <Greeting /> {/* No name prop, uses "Guest" fallback */}

            <h2>Products:</h2>
            <div style={{ display: 'flex', gap: '20px', flexWrap: 'wrap' }}>
                {/* Mapping over an array to render multiple ProductCard components */}
                {products.map((product, index) => (
                    <ProductCard
                        key={index} // Unique key for list items
                        productName={product.productName}
                        price={product.price}
                        isInStock={product.isInStock}
                    />
                ))}
            </div>

            <ShoppingList items={shoppingItems} />
            <ShoppingList /> {/* Renders an empty list message */}

            <h2>Fragment Example:</h2>
            <MyFragmentExample />
        </div>
    );
}

// Typically, in a React app, you would render the App component to the DOM.
// Example (conceptual, requires ReactDOM):
// import ReactDOM from 'react-dom/client';
// const root = ReactDOM.createRoot(document.getElementById('root'));
// root.render(<App />);

// For demonstration purposes without a full React setup, we'll just show the structure.
// If this were transpiled and run, the output would be HTML elements.
console.log("JSX is a syntax extension. This code would be transpiled by Babel.");
console.log("For example, <h1>Hello!</h1> might become React.createElement('h1', null, 'Hello!');");

// You can't directly execute JSX without a transpiler.
// The concepts above show how it looks and how JavaScript is integrated within it.
```
# **Flashcards:**
---
What is JSX?;; A syntax extension for JavaScript that allows writing HTML-like code within JavaScript, primarily for defining UI structures. 

How is JSX processed by web browsers?;; It's not directly understood; it must be "transpiled" into regular JavaScript function calls (e.g., `React.createElement()`) by a tool like Babel before the browser can execute it. 

What is a key difference between JSX and standard HTML attributes?;; JSX uses camelCase for attributes like `className` and `onClick`, whereas HTML uses lowercase `class` and `onclick`. Also, self-closing tags must be explicitly closed (e.g., `<img />`).