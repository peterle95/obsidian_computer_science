---
memory: to_finish
tags:
  - to_learn
language:
  - Core Concepts
review-date: 2025-11-20
last-reviewed: 2025-10-20
keywords:
scheda: done
visit-count: 2
confidence-level: 1
consecutive-correct: 0
last-struggle-date: 2025-10-20
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

The fundamental problem the wrapper concept solves is the need to ==**extend, modify, or simplify the functionality of existing code without altering its source code**.== Wrappers act as an intermediary, enclosing a piece of code (like a class, function, or primitive type) to provide a different interface or added capabilities.

This is important in computer science because it promotes **code reusability, flexibility, and maintainability**. It allows developers to integrate incompatible components, add features dynamically, and simplify complex systems. The concept is a cornerstone of many design patterns that are widely used in software engineering.

# **Core Explanation:**
---

## Wrapper in the JavaScript Context

In JavaScript, primitive values (like true or false) don't have methods or properties by themselves. ==A wrapper is an object that "wraps" around a primitive value to provide object-like functionality.== JavaScript has wrapper objects for primitive types:

- Boolean for boolean primitives
- String for string primitives
- Number for number primitives
- BigInt for bigint primitives
- Symbol for symbol primitives

==When you try to access a property or method on a primitive value, JavaScript temporarily creates a wrapper object, accesses the property/method, and then discards the wrapper. This happens automatically.==

## Wrappers in Broader Computer Science/Programming Context

In the broader context of computer science and programming languages, a wrapper has several related meanings:

1. **Primitive Type Wrappers**: As seen in JavaScript, Java, and other languages, these are objects that encapsulate primitive data types to provide object-oriented functionality.

2. **Function/Method Wrappers**: A pattern where a function is enclosed within another function to add functionality like logging, error handling, caching, or rate limiting without modifying the original function.

3. **API Wrappers**: Libraries that simplify or adapt complex APIs into more user-friendly interfaces. They "wrap" around existing code to make it easier to use.

4. **Class Wrappers**: Classes that encapsulate other classes or data structures to provide additional functionality or a different interface.

5. **Adapter Pattern**: A design pattern where a wrapper translates one interface to another, allowing incompatible interfaces to work together.

6. **Facade Pattern**: A design pattern that provides a simplified interface to a complex subsystem.

7. **Library/Module Wrappers**: Code that encapsulates external libraries to standardize their usage within a project or to isolate dependencies.


The core concept across all these uses is encapsulation - a wrapper "wraps around" something else to extend, modify, or adapt its behavior while maintaining its core functionality.

## **Usage in Programming Languages and Modernity**
---
## **In which languages is it used?**

The Wrapper concept is not tied to a specific language; it is a fundamental design principle. As such, it is used in virtually all modern programming languages, including:

- **Object-Oriented Languages:** Java, C# , C++, Python, and Ruby heavily use wrapper patterns like Decorator, Adapter, and Facade. Java's standard library includes wrapper classes for all its primitive types (e.g., Integer for int, Boolean for boolean).

- **Functional Languages:** In languages that support higher-order functions, function wrappers (often called decorators in Python) are a common idiom.

- **JavaScript:** As mentioned, JavaScript uses wrappers for primitive types and the concept is widely applied in libraries and frameworks to create more manageable interfaces for complex APIs.

- **Web Development:** In HTML and CSS, a "wrapper" div is often used to contain and style a group of elements.

## **Is it a modern tool?**

While the underlying principles of wrapping are part of foundational software design patterns that have been around for decades (like those in the "Gang of Four" Design Patterns book), the wrapper concept remains a **timeless and modern tool**. Its applications are highly relevant in contemporary software development for several reasons:

- **API Integration:** Modern applications frequently rely on numerous third-party APIs. ==API wrappers are crucial for simplifying these integrations and insulating the application from changes in the external APIs.==

- **Legacy Code:** <mark style="background: #BBFABBA6;">Wrappers are an essential technique for modernizing and working with legacy systems without extensive rewriting.</mark> An adapter can make old code compatible with new systems.

- **Complexity Management:** As software systems grow in complexity, the Facade pattern (a type of wrapper) is a modern approach to providing simple, understandable entry points to complex subsystems.

- **Cross-language Interoperability:** <mark style="background: #FF5582A6;">Wrappers can bridge the gap between different programming languages, allowing code from one language to be used in another</mark>.


In essence, while the concept is not new, its practical applications are continually adapted to solve modern software engineering challenges.

# **Related Concepts:**
---

- **Adapter Pattern:** This pattern acts as a bridge between two incompatible interfaces. It wraps an existing class with a new interface so that it can work with other classes. The key focus is on **translating one interface to another**.

- **Decorator Pattern:** This pattern allows behavior to be added to an individual object, either statically or dynamically, without affecting the behavior of other objects from the same class. It wraps an object to provide **extended functionality**.

- **Facade Pattern:** This pattern provides a simplified, higher-level interface to a larger body of code, such as a complex subsystem of classes. It wraps multiple interfaces into a single, more convenient one. Its goal is **simplification**.

- **Proxy Pattern:** This pattern provides a surrogate or placeholder for another object to control access to it. It is often used for lazy initialization, access control, logging, or monitoring. The wrapper in this case **controls access** to the original object.

- **Composition:** This is the underlying principle for most wrapper patterns. It refers to building complex objects by "composing" them from other objects, as opposed to inheriting from a base class. Wrappers typically hold an instance of the object they are wrapping.

# **Examples:**
---
###### JavaScript: API Wrapper for a basic Fetch request

```js
// This wrapper simplifies making GET requests with the fetch API
// by handling the conversion to JSON and basic error handling.

class ApiWrapper {
    constructor(baseUrl) {
        this.baseUrl = baseUrl;
    }

    // The 'get' method wraps the standard 'fetch' call.
    async get(endpoint) {
        try {
            // It constructs the full URL and makes the request.
            const response = await fetch(`${this.baseUrl}/${endpoint}`);

            // It includes error handling for bad responses.
            if (!response.ok) {
                throw new Error(`HTTP error! status: ${response.status}`);
            }

            // It handles the conversion of the response to JSON.
            const data = await response.json();
            return data;

        } catch (error) {
            // It simplifies error logging for the client.
            console.error("API request failed:", error);
            return null;
        }
    }
}

// --- Client Usage ---

// Create an instance of the wrapper for a specific API.
const jsonPlaceholderApi = new ApiWrapper('https://jsonplaceholder.typicode.com');

// The client now uses the simple 'get' method without needing to
// know the details of fetch, response.ok, or response.json().
jsonPlaceholderApi.get('todos/1').then(data => {
    if (data) {
        console.log("Fetched data:", data);
    }
});
```
### Python: Function Wrapper (Decorator) for Logging

```python
# A decorator is a function that takes another function as an argument,
# adds some kind of functionality, and then returns another function.
# This is a classic example of a wrapper.

def logger_wrapper(func):
    # This is the wrapper function that will be returned.
    # It takes the same arguments as the original function.
    def wrapper(*args, **kwargs):
        # Action before the original function is called.
        print(f"--- Calling function '{func.__name__}' with arguments {args} and {kwargs} ---")
        
        # Call the original function.
        result = func(*args, **kwargs)
        
        # Action after the original function is called.
        print(f"--- Function '{func.__name__}' returned: {result} ---")
        
        return result
    return wrapper

# Using the decorator to wrap the 'add' function.
@logger_wrapper
def add(a, b):
    """This function adds two numbers."""
    return a + b

# Calling the wrapped function.
add(5, 3)
```
# **Flashcards:**
---

**What is a Wrapper in programming?**;;A wrapper is a design principle where a class, function, or other entity encapsulates, or "wraps," another item to extend or modify its behavior without changing the original item's source code. It acts as an intermediary or a layer of abstraction.

**What is the key difference between the Adapter and Decorator wrapper patterns?**;;The Adapter pattern's main purpose is to **change an interface** to make it compatible with another, while the Decorator pattern's purpose is to **add responsibilities** or functionality to an object dynamically.

**Why is new Boolean(false) problematic in JavaScript?**;;Because while the primitive value it wraps is false, the new Boolean(false) expression creates an **object**. In JavaScript, all objects are "truthy," meaning they evaluate to true in a boolean context (like an if statement). This can lead to unexpected behavior.
