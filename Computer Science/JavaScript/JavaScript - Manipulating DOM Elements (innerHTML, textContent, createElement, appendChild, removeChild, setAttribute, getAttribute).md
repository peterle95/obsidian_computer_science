---
memory: to_finish
tags:
  - learned
language:
  - JavaScript
review-date:
last-reviewed: 2025-09-03
scheda: done
visit-count: 4
confidence-level: 2.5
consecutive-correct: 3
last-struggle-date: 2025-08-01
cssclasses: []
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

DOM manipulation is the cornerstone of interactive web development. It solves the fundamental problem of creating dynamic web pages that can respond to user interactions and update content without requiring page reloads. Before DOM manipulation, web pages were static documents. This concept enables developers to create rich, interactive user interfaces by programmatically modifying HTML elements, their content, attributes, and structure in real-time. It's essential because it bridges the gap between static HTML markup and dynamic, responsive web applications.

# **Core Explanation:**
---

DOM (Document Object Model) manipulation refers to the programmatic modification of HTML elements and their properties using JavaScript. The DOM represents the HTML document as a tree structure where each HTML element is a node that can be accessed, modified, created, or removed.

Key methods for DOM manipulation:

>- **innerHTML**: Gets or sets the HTML content inside an element, <mark style="background: #FF5582A6;">including HTML tags</mark>
>- **textContent**: Gets or sets only the text content, <mark style="background: #FF5582A6;">stripping HTML tags</mark>
>- **createElement()**: Creates a n<mark style="background: #ADCCFFA6;">ew HTML element node</mark>
>- **appendChild()**:<mark style="background: #D2B3FFA6;"> Adds a child element to a parent element</mark>
>- **removeChild()**: <mark style="background: #D2B3FFA6;">Removes a child element from its parent</mark>
>- **setAttribute()**: Sets an attribute value on an element
>- **getAttribute()**: Retrieves an attribute value from an element

These methods work by accessing elements through selectors (getElementById, querySelector, etc.) and then modifying their properties or structure. The browser automatically updates the visual representation when DOM changes occur.

# **Related Concepts:**
---

- **DOM Selection Methods**: querySelector(), getElementById(), getElementsByClassName() - These are prerequisite methods for selecting elements before manipulation
- **Event Handling**: addEventListener() - Often used together with DOM manipulation to create interactive responses
- **CSS Manipulation**: style property, classList methods - Closely related as they modify element appearance rather than content/structure
- **Document Fragments**: createDocumentFragment() - Optimized way to batch DOM modifications
- **Node Properties**: parentNode, childNodes, nextSibling - Navigate and understand DOM relationships
- **jQuery**: A library that simplifies DOM manipulation with methods like .html(), .text(), .append()
- **Virtual DOM**: React's concept that optimizes DOM manipulation by creating a virtual representation

# **Examples:**
---

```javascript
// Example 1: innerHTML vs textContent
const container = document.getElementById('container');

// innerHTML preserves HTML tags and renders them
container.innerHTML = '<strong>Bold text</strong> and normal text';

// textContent treats everything as plain text, no HTML rendering
container.textContent = '<strong>This appears as literal text</strong>';

// Example 2: Creating and adding elements dynamically
// Create a new paragraph element in memory
const newParagraph = document.createElement('p');

// Set text content for the new paragraph
newParagraph.textContent = 'This is a dynamically created paragraph';

// Set attributes on the new element
newParagraph.setAttribute('class', 'dynamic-content');
newParagraph.
setAttribute('id', 'new-para');

// Find parent element and append the new paragraph to it
const parentDiv = document.querySelector('.content-area');
parentDiv.appendChild(newParagraph);

// Example 3: Modifying existing elements
const existingElement = document.getElementById('existing-para');

// Get current attribute value
const currentClass = existingElement.getAttribute('class');
console.log('Current class:', currentClass);

// Modify the element's attributes
existingElement.setAttribute('data-modified', 'true');
existingElement.setAttribute('style', 'color: blue; font-weight: bold;');

// Change the content
existingElement.innerHTML = 'This content has been <em>modified</em>!';

// Example 4: Removing elements
const elementToRemove = document.getElementById('remove-me');
const parent = elementToRemove.parentNode;

// Remove the element from its parent
parent.removeChild(elementToRemove);

// Alternative modern approach (newer browsers)
// elementToRemove.remove();

// Example 5: Building a dynamic list
const listContainer = document.getElementById('dynamic-list');
const items = ['Apple', 'Banana', 'Cherry', 'Date'];

// Create unordered list element
const ul = document.createElement('ul');
ul.setAttribute('class', 'fruit-list');

// Loop through items and create list items
items.forEach((item, index) => {
    // Create list item element
    const li = document.createElement('li');
    
    // Set text content and attributes
    li.textContent = item;
    li.setAttribute('data-index', index);
    li.setAttribute('class', 'fruit-item');
    
    // Add click event listener for interactivity
    li.addEventListener('click', function() {
        // When clicked, highlight the item
        this.style.backgroundColor = 'yellow';
        console.log('Clicked:', this.textContent);
    });
    
    // Append list item to unordered list
    ul.appendChild(li);
});

// Append the complete list to the container
listContainer.appendChild(ul);

// Example 6: Form manipulation and validation
const form = document.getElementById('user-form');
const nameInput = document.getElementById('name-input');
const submitBtn = document.getElementById('submit-btn');

// Create error message element
const errorDiv = document.createElement('div');
errorDiv.setAttribute('class', 'error-message');
errorDiv.style.color = 'red';
errorDiv.style.display = 'none';

// Insert error message after the input field
nameInput.parentNode.insertBefore(errorDiv, nameInput.nextSibling);

// Add real-time validation
nameInput.addEventListener('input', function() {
    const value = this.value.trim();
    
    if (value.length < 2) {
        // Show error message
        errorDiv.textContent = 'Name must be at least 2 characters';
        errorDiv.style.display = 'block';
        
        // Disable submit button
        submitBtn.setAttribute('disabled', 'true');
        submitBtn.textContent = 'Please fix errors';
    } else {
        // Hide error message
        errorDiv.style.display = 'none';
        
        // Enable submit button
        submitBtn.removeAttribute('disabled');
        submitBtn.textContent = 'Submit';
    }
});
```
# Flashcards:
-----

What is the difference between innerHTML and textContent?;; innerHTML gets/sets HTML content including tags and renders them, while textContent gets/sets only plain text content and treats HTML as literal text

How do you create a new HTML element in JavaScript?;; Use document.createElement(‘tagName’) to create a new element node in memory, then use appendChild() or insertBefore() to add it to the DOM

What method removes a child element from its parent?;; parentNode.removeChild(childElement) removes the specified child element from its parent node

How do you set an attribute on a DOM element?;; Use element.setAttribute(‘attributeName’, ‘value’) to set an attribute, or element.getAttribute(‘attributeName’) to retrieve an attribute value

What happens when you use appendChild() on an element?;; appendChild() adds the specified element as the last child of the parent element, and the browser automatically updates the visual representation

Name three methods for selecting DOM elements before manipulation;; document.getElementById(‘id’), document.querySelector(‘selector’), and document.getElementsByClassName(‘className’) are common selection methods