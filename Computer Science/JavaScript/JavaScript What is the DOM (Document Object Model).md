---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date:
last-reviewed: 2025-08-28
scheda: done
visit-count: 4
confidence-level: 2
consecutive-correct: 3
last-struggle-date: 2025-07-22

---
```dataviewjs
const currentPage = dv.current;
let visitCount = currentPage.file.frontmatter["visit-count"] || 0;
let confidence = currentPage.file.frontmatter["confidence-level"] || 1;
let streak = currentPage.file.frontmatter["consecutive-correct"] || 0;

const container = this.container.createEl('div');
container.style.cssText = `
 background:

# 2a2a2a; border: 1px solid

# 404040; border-radius: 6px;
 padding: 12px; margin: 10px 0; display: inline-block;
`;

// Status display
const status = container.createEl('div');
status.innerHTML = `
 <strong>Learning Progress</strong><br>
 Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
`;
status.style.cssText = 'margin-bottom: 10px; font-size: 13px; color:

# cccccc;';

// Quick feedback buttons
const buttonContainer = container.createEl('div');
['Got it! ✅', 'Struggled ⚠️', 'Failed ❌'].forEach((label, index) => {
 const btn = buttonContainer.createEl('button');
 btn.textContent = label;
 btn.style.cssText = `
 margin-right: 8px; padding: 4px 8px; border: none; border-radius: 3px;
 cursor: pointer; font-size: 11px;
 background: ${['

# 28a745', '

# ffc107', '

# dc3545'][index]};
 color: ${index === 1 ? '

# 000' : '

# fff'};
 `;

 btn.addEventListener('click', async => {
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
 fm["last-reviewed"] = new Date.toISOString.split('T');
 if (index > 0) fm["last-struggle-date"] = new Date.toISOString.split('T');
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
const currentPage = dv.current;
const content = await app.vault.read(app.vault.getAbstractFileByPath(currentPage.file.path));

// Split content into lines
const lines = content.split('\n');
let flashcardLines = ;
let inCodeBlock = false;

// Collect all potential flashcard lines - simplified approach
for (let i = 0; i < lines.length; i++) {
 const line = lines[i];

 // Track code blocks
 if (line.trim.startsWith('```')) {
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
 line.trim.startsWith('const ') ||
 line.trim.startsWith('let ') ||
 line.trim.startsWith('function ') ||
 line.trim.startsWith('return ') ||
 line.trim.startsWith('if (') ||
 line.trim.startsWith('for (') ||
 line.trim.startsWith('while (') ||
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
const flashcards = ;
for (let i = 0; i < filteredLines.length; i++) {
 const line = filteredLines[i];
 try {
 const separatorIndex = line.indexOf(';;');
 if (separatorIndex === -1) continue;

 const front = line.substring(0, separatorIndex).trim;
 const back = line.substring(separatorIndex + 2).trim;

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
 errorMsg.style.cssText = 'background:

# 2a2a2a; padding: 15px; border-radius: 6px; color:

# cccccc;';
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
 background:

# 2a2a2a;
 border: 1px solid

# 404040;
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
title.style.cssText = 'margin: 0; color:

# ffffff;';

const progress = header.createEl('div');
progress.style.cssText = 'color:

# cccccc; font-size: 14px; text-align: right;';

// Card container
const cardContainer = container.createEl('div');
cardContainer.style.cssText = `
 background:

# 1a1a1a;
 border: 2px solid

# 404040;
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
 color:

# ffffff;
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
 background:

# 4a9eff; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; font-weight: 500;
`;

const easyButton = buttonContainer.createEl('button');
easyButton.textContent = 'Easy ✅';
easyButton.style.cssText = `
 background:

# 28a745; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const hardButton = buttonContainer.createEl('button');
hardButton.textContent = 'Hard ❌';
hardButton.style.cssText = `
 background:

# dc3545; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const nextButton = buttonContainer.createEl('button');
nextButton.textContent = 'Next →';
nextButton.style.cssText = `
 background:

# 6c757d; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const prevButton = buttonContainer.createEl('button');
prevButton.textContent = '← Prev';
prevButton.style.cssText = `
 background:

# 6c757d; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const shuffleButton = buttonContainer.createEl('button');
shuffleButton.textContent = '🔀 Shuffle';
shuffleButton.style.cssText = `
 background:

# 17a2b8; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

// Functions
function updateDisplay {
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
 cardContainer.style.borderColor = '

# ffc107';
 cardContainer.style.backgroundColor = '

# 252525';
 } else {
 easyButton.style.display = 'none';
 hardButton.style.display = 'none';
 flipButton.textContent = 'Flip Card';
 cardContainer.style.borderColor = '

# 404040';
 cardContainer.style.backgroundColor = '

# 1a1a1a';
 }

 // Update navigation buttons
 prevButton.style.display = currentCardIndex > 0 ? 'inline-block' : 'none';
 nextButton.textContent = currentCardIndex < flashcards.length - 1 ? 'Next →' : 'Restart';
}

function flipCard {
 showingBack = !showingBack;
 updateDisplay;
}

function nextCard {
 if (currentCardIndex < flashcards.length - 1) {
 currentCardIndex++;
 } else {
 currentCardIndex = 0;
 }
 showingBack = false;
 updateDisplay;
}

function prevCard {
 if (currentCardIndex > 0) {
 currentCardIndex--;
 showingBack = false;
 updateDisplay;
 }
}

function markCorrect {
 if (showingBack) {
 correctCount++;
 totalReviewed++;
 nextCard;
 }
}

function markIncorrect {
 if (showingBack) {
 totalReviewed++;
 nextCard;
 }
}

function shuffleCards {
 for (let i = flashcards.length - 1; i > 0; i--) {
 const j = Math.floor(Math.random * (i + 1));
 [flashcards[i], flashcards[j]] = [flashcards[j], flashcards[i]];
 }
 currentCardIndex = 0;
 showingBack = false;
 correctCount = 0;
 totalReviewed = 0;
 updateDisplay;
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
instructions.style.cssText = 'font-size: 12px; color:

# 888; text-align: center; line-height: 1.4;';
instructions.innerHTML = `
 <strong>Controls:</strong> Click card to flip | Navigation buttons | Easy/Hard to mark
`;

// Initialize
updateDisplay;
```

# **Purpose/Why**:

---
The DOM (Document Object Model) solves the fundamental problem of how programming languages can ==interact with and manipulate web documents dynamically. It provides a structured, programmable interface that represents HTML and XML documents as a tree of objects, allowing JavaScript to access, modify, create, and delete elements and their content in real-time==. This is crucial for creating interactive web applications because it bridges the gap between static markup and dynamic functionality. ==Without the DOM, web pages would be static documents with no ability to respond to user interaction==s, update content dynamically, or create rich user experiences. The DOM is essential for modern web development as it enables everything from simple form validation to complex single-page applications.

# **Core Explanation:**

---

The DOM (Document Object Model) is a programming interface for web documents that <mark style="background:

# ABF7F7A6;">represents the page as a structured tree of objects. Each HTML element, attribute, and piece of text becomes a node in this tree, which can be accessed and manipulated through JavaScript.</mark>

Key characteristics include:
- **Tree Structure**: Documents are represented as a hierarchical tree with parent-child relationships
- **Node-Based**: Every element, attribute, and text content is represented as a node object
- **Live Representation**: Changes to the DOM immediately reflect in the rendered page
- **Language-Agnostic**: While commonly used with JavaScript, the DOM is designed to work with any programming language
- **Standardized API**: Provides consistent methods and properties across different browsers
- **Event-Driven**: Supports event handling for user interactions and document changes

The DOM works by parsing HTML markup and creating a tree structure where each element becomes an object with properties and methods. JavaScript can traverse this tree, select elements, modify their content, attributes, and styles, add or remove elements, and attach event listeners. When the DOM is modified, the browser automatically updates the visual representation of the page, creating dynamic and interactive web experiences.

JavaScript interacts with the HTML using the Document Object Model, or DOM. The DOM is a tree of objects that represents the HTML. You can access the HTML using the `document` object, which represents your entire HTML document.

One method for finding specific elements in your HTML is using the `querySelector` method. The `querySelector` method takes a CSS selector as an argument and returns the first element that matches that selector. For example, to find the `<h1>` element in your HTML, you would write:

```js
let h1 = document.querySelector("h1");
```

Note that `h1` is a string and matches the CSS selector you would use.

# **Related Concepts:**

---
- **HTML Elements**: The building blocks that the DOM represents as objects, providing the structure that gets converted into DOM nodes
- **CSS Styling**: Works with the DOM to control presentation; JavaScript can manipulate CSS through DOM properties
- **Event Handling**: DOM events (click, hover, submit) allow JavaScript to respond to user interactions
- **Browser APIs**: The DOM is part of the Web APIs provided by browsers for JavaScript interaction
- **Virtual DOM**: Used by frameworks like React as an abstraction layer over the real DOM for performance optimization
- **DOM Manipulation vs jQuery**: jQuery provides a simplified syntax for DOM operations, but modern JavaScript has similar capabilities
- **Shadow DOM**: Encapsulated DOM trees that provide component isolation, part of Web Components specification
- **Document Parsing**: The browser process that converts HTML text into the DOM tree structure
- **Rendering Engine**: The browser component that uses the DOM to paint the visual representation of the page

# **Examples:**

---
```html
<!DOCTYPE html>
<html lang="en">
<head>
 <meta charset="UTF-8">
 <meta name="viewport" content="width=device-width, initial-scale=1.0">
 <title>DOM Examples</title>
 <style>
 .highlight { background-color: yellow; }
 .hidden { display: none; }
 .card { border: 1px solid

# ccc; padding: 10px; margin: 5px; }
 </style>
</head>
<body>
 <h1 id="main-title">DOM Manipulation Examples</h1>

 <div class="container">
 <p id="demo-paragraph">This is a paragraph that will be modified.</p>
 <button id="change-text-btn">Change Text</button>
 <button id="toggle-highlight-btn">Toggle Highlight</button>
 </div>

 <div class="form-section">
 <h2>Dynamic Form</h2>
 <input type="text" id="user-input" placeholder="Enter your name">
 <button id="add-item-btn">Add Item</button>
 <ul id="items-list"></ul>
 </div>

 <div class="card-section">
 <h2>Dynamic Cards</h2>
 <button id="create-card-btn">Create Card</button>
 <div id="cards-container"></div>
 </div>

 <script>
 // 1. SELECTING DOM ELEMENTS
 // Different methods to select elements from the DOM tree

 // Select by ID - returns single element or null
 const title = document.getElementById('main-title');
 const paragraph = document.getElementById('demo-paragraph');

 // Select by class name - returns HTMLCollection (live collection)
 const containers = document.getElementsByClassName('container');

 // Select by tag name - returns HTMLCollection
 const allButtons = document.getElementsByTagName('button');

 // Modern selector methods - work like CSS selectors
 const changeTextBtn = document.querySelector('

# change-text-btn'); // First match
 const allCards = document.querySelectorAll('.card'); // All matches (NodeList)

 console.log('Title element:', title);
 console.log('All buttons:', allButtons);

 // 2. READING AND MODIFYING ELEMENT CONTENT
 // Different ways to access and change element content

 // textContent - gets/sets text content without HTML tags
 console.log('Original paragraph text:', paragraph.textContent);

 // innerHTML - gets/sets HTML content (can include tags)
 console.log('Original paragraph HTML:', paragraph.innerHTML);

 // innerText - gets/sets visible text (respects styling like display:none)
 console.log('Original paragraph visible text:', paragraph.innerText);

 // 3. MODIFYING ELEMENT ATTRIBUTES AND PROPERTIES
 // Working with element attributes and CSS properties

 function changeTextContent {
 // Modify text content of the paragraph
 paragraph.textContent = 'This text has been changed by JavaScript!';

 // Modify attributes
 paragraph.setAttribute('data-modified', 'true');
 paragraph.title = 'This paragraph was modified'; // Direct property access

 // Modify CSS styles directly
 paragraph.style.color = 'blue';
 paragraph.style.fontSize = '18px';

 console.log('Text content modified');
 }

 function toggleHighlight {
 // Toggle CSS class - demonstrates className and classList
 if (paragraph.classList.contains('highlight')) {
 paragraph.classList.remove('highlight');
 console.log('Highlight removed');
 } else {
 paragraph.classList.add('highlight');
 console.log('Highlight added');
 }

 // Alternative using className (less convenient)
 // paragraph.className = paragraph.className.includes('highlight') ?
 // paragraph.className.replace('highlight', '').trim :
 // paragraph.className + ' highlight';
 }

 // 4. CREATING AND ADDING NEW ELEMENTS
 // Dynamically creating DOM elements and adding them to the tree

 function addListItem {
 const userInput = document.getElementById('user-input');
 const itemsList = document.getElementById('items-list');

 // Get input value
 const inputValue = userInput.value.trim;

 if (inputValue === '') {
 alert('Please enter a name');
 return;
 }

 // Create new DOM element
 const newListItem = document.createElement('li');

 // Set content and attributes
 newListItem.textContent = inputValue;
 newListItem.setAttribute('data-created', new Date.toISOString);

 // Create delete button for the item
 const deleteBtn = document.createElement('button');
 deleteBtn.textContent = 'Delete';
 deleteBtn.style.marginLeft = '10px';
 deleteBtn.style.backgroundColor = '

# ff4444';
 deleteBtn.style.color = 'white';
 deleteBtn.style.border = 'none';
 deleteBtn.style.padding = '2px 8px';
 deleteBtn.style.cursor = 'pointer';

 // Add event listener to delete button
 deleteBtn.addEventListener('click', function {
 // Remove the list item from the DOM
 itemsList.removeChild(newListItem);
 console.log('Item removed:', inputValue);
 });

 // Append delete button to list item
 newListItem.appendChild(deleteBtn);

 // Add the new item to the list
 itemsList.appendChild(newListItem);

 // Clear the input field
 userInput.value = '';

 console.log('Added new item:', inputValue);
 }

 // 5. ADVANCED DOM MANIPULATION
 // More complex DOM operations and traversal

 function createDynamicCard {
 const cardsContainer = document.getElementById('cards-container');

 // Create card structure
 const card = document.createElement('div');
 card.className = 'card';

 // Create card content using innerHTML for complex structure
 const cardId = Date.now; // Simple unique ID
 card.innerHTML = `
 <h3>Card

# ${cardId}</h3>
 <p>This card was created at ${new Date.toLocaleTimeString}</p>
 <button class="edit-btn" data-card-id="${cardId}">Edit</button>
 <button class="delete-btn" data-card-id="${cardId}">Delete</button>
 `;

 // Add event listeners using event delegation
 card.addEventListener('click', function(event) {
 const target = event.target;

 if (target.classList.contains('edit-btn')) {
 // Edit functionality
 const cardTitle = card.querySelector('h3');
 const newTitle = prompt('Enter new title:', cardTitle.textContent);
 if (newTitle) {
 cardTitle.textContent = newTitle;
 }
 } else if (target.classList.contains('delete-btn')) {
 // Delete functionality with confirmation
 if (confirm('Are you sure you want to delete this card?')) {
 // Remove card with animation
 card.style.transition = 'opacity 0.3s ease';
 card.style.opacity = '0';
 setTimeout( => {
 cardsContainer.removeChild(card);
 }, 300);
 }
 }
 });

 // Add card to container
 cardsContainer.appendChild(card);

 console.log('Created new card with ID:', cardId);
 }

 // 6. DOM TRAVERSAL
 // Navigating through the DOM tree structure

 function demonstrateDOMTraversal {
 const container = document.querySelector('.container');

 // Parent node traversal
 console.log('Container parent:', container.parentNode);
 console.log('Container parent element:', container.parentElement);

 // Child node traversal
 console.log('Container children:', container.children); // Only element nodes
 console.log('Container child nodes:', container.childNodes); // All nodes including text
 console.log('First child element:', container.firstElementChild);
 console.log('Last child element:', container.lastElementChild);

 // Sibling traversal
 const firstChild = container.firstElementChild;
 if (firstChild) {
 console.log('Next sibling:', firstChild.nextElementSibling);
 console.log('Previous sibling:', firstChild.previousElementSibling);
 }
 }

 // 7. EVENT HANDLING
 // Attaching event listeners to DOM elements

 // Method 1: Using addEventListener (recommended)
 changeTextBtn.addEventListener('click', changeTextContent);

 document.getElementById('toggle-highlight-btn').addEventListener('click', toggleHighlight);

 document.getElementById('add-item-btn').addEventListener('click', addListItem);

 document.getElementById('create-card-btn').addEventListener('click', createDynamicCard);

 // Method 2: Using event properties (less flexible)
 // changeTextBtn.onclick = changeTextContent;

 // 8. DOCUMENT READY AND WINDOW EVENTS
 // Handling document lifecycle events

 // DOMContentLoaded - fires when DOM is fully constructed
 document.addEventListener('DOMContentLoaded', function {
 console.log('DOM fully loaded and parsed');
 demonstrateDOMTraversal;
 });

 // Window load - fires when all resources are loaded
 window.addEventListener('load', function {
 console.log('Page fully loaded including images and stylesheets');
 });

 // 9. FORM HANDLING
 // Working with form elements and input

 const userInput = document.getElementById('user-input');

 // Listen for Enter key in input field
 userInput.addEventListener('keypress', function(event) {
 if (event.key === 'Enter') {
 addListItem;
 }
 });

 // Input validation example
 userInput.addEventListener('input', function {
 const value = this.value;

 // Real-time validation
 if (value.length > 20) {
 this.style.borderColor = 'red';
 this.title = 'Name is too long (max 20 characters)';
 } else {
 this.style.borderColor = '';
 this.title = '';
 }
 });

 // 10. PERFORMANCE CONSIDERATIONS
 // Demonstrating efficient DOM manipulation

 function efficientDOMUpdate {
 const container = document.getElementById('cards-container');

 // Inefficient way - multiple DOM updates
 // for (let i = 0; i < 100; i++) {
 // const div = document.createElement('div');
 // div.textContent = `Item ${i}`;
 // container.appendChild(div); // Each append triggers reflow
 // }

 // Efficient way - batch DOM updates
 const fragment = document.createDocumentFragment;
 for (let i = 0; i < 100; i++) {
 const div = document.createElement('div');
 div.textContent = `Efficient Item ${i}`;
 fragment.appendChild(div); // Append to fragment, not DOM
 }
 container.appendChild(fragment); // Single DOM update

 console.log('Efficient DOM update completed');
 }

 // Add button to test efficient updates
 const efficientBtn = document.createElement('button');
 efficientBtn.textContent = 'Add 100 Items Efficiently';
 efficientBtn.addEventListener('click', efficientDOMUpdate);
 document.body.appendChild(efficientBtn);

 </script>
</body>
</html>
````

# **Flashcards:**

---
What is the DOM and how does it represent web documents?;; The DOM (Document Object Model) is a programming interface that represents HTML/XML documents as a structured tree of objects, where each element, attribute, and text content becomes a node that can be accessed and manipulated programmatically.

What are the main methods for selecting DOM elements in JavaScript?;; getElementById for single elements by ID, getElementsByClassName and getElementsByTagName for collections, querySelector for single elements using CSS selectors, and querySelectorAll for multiple elements using CSS selectors.

What is the difference between textContent, innerHTML, and innerText?;; textContent gets/sets plain text without HTML tags, innerHTML gets/sets HTML content including tags, and innerText gets/sets visible text while respecting CSS styling like display:none.

How do you create and add new elements to the DOM?;; Use document.createElement to create elements, set properties/content, then use appendChild, insertBefore, or similar methods to add them to the DOM tree. Document fragments can be used for efficient batch operations.

What is event delegation and why is it useful in DOM manipulation?;; Event delegation is attaching a single event listener to a parent element to handle events from child elements using event bubbling. It's useful for handling events on dynamically created elements and improving performance.

What are the performance considerations when manipulating the DOM?;; Minimize DOM queries by caching element references, batch DOM updates using document fragments, avoid frequent style changes that trigger reflows, and use efficient selectors. Each DOM modification can trigger expensive reflow/repaint operations.