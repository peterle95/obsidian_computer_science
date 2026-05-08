---
memory: to_finish
tags:
  - mastered
language:
  - C++
review-date: ""
last-reviewed: 2025-09-01
scheda: done
visit-count: 5
confidence-level: 3.5
consecutive-correct: 5
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
# **Core Explanation:**

In C++ [[C++ - Exception Handling]], the `catch` keyword is used to define a block of code, known as an **exception handler**, that executes when a specific type of exception is thrown within the preceding `try` block.

## Purpose

The primary purpose of a `catch` block is to "catch" and handle exceptions that have been `thrown`. It allows the program to gracefully respond to error conditions detected in the `try` block, rather than terminating abruptly. By handling the exception, the program can potentially recover, log the error, or terminate in a more controlled manner.

## Syntax

A `catch` block must immediately follow a `try` block or another `catch` block associated with the same `try`. A single `try` block can be followed by multiple `catch` blocks to handle different types of exceptions.

```cpp
try {
  // Code that might throw an exception
  // ...
  // throw SomeExceptionType(...);
  // ...
}
catch (ExceptionType1 param1) { // Catches exceptions of Type1
  // Handler code for ExceptionType1
  // 'param1' holds the thrown exception object/value
}
catch (ExceptionType2& param2) { // Catches exceptions of Type2 (often by const reference)
  // Handler code for ExceptionType2
  // 'param2' refers to the thrown exception object
}
catch (AnotherType* param3) { // Catches pointer exceptions (less common for errors)
    // Handler code for pointers of AnotherType
    // 'param3' points to the thrown object/value
}
// ... potentially more specific catch blocks ...
catch (...) { // Catch-all block (ellipsis)
  // Handler code for any exception type not caught above
  // Cannot access the exception object directly here
}
```

- `ExceptionType`: Specifies the type of exception this block can handle.

- `param`: An optional parameter name. If provided, it receives a copy of, a reference to, or a pointer to the thrown exception object/value, allowing the handler code to access information about the error. Catching by const reference (const ExceptionType&) is generally preferred for class types to avoid copying and slicing issues.

### How it Works

- `Exception Thrown`: When an exception is thrown inside a [[try]] block, the program immediately stops executing code within the try block.

- `Handler Matching`: The runtime searches the sequence of catch blocks associated with that try block in the order they appear.

- `First Match Wins`: The first catch block whose `ExceptionType` matches the type of the thrown exception (or is a public base class of the thrown exception type, or handles any type via ...) is selected.

### Execution: The code inside the selected catch block is executed.

- `Continuation`: After the catch block finishes executing, program execution continues with the statement immediately following the last catch block in the sequence (unless the catch block itself throws another exception or terminates the program).

- `No Match`: If no matching catch block is found for the thrown exception, the exception propagates up the call stack, potentially terminating the program if it remains unhandled (often via std::terminate).

## Exception Types

C++ allows exceptions of virtually any type to be thrown:

1. `Primitive Types`: As seen in the original example, you can throw and catch fundamental types like int, double, char* (C-style strings), etc.

```cpp
try {
    throw 404; // Throw an integer
} catch (int errorCode) {
    std::cerr << "Error occurred: " << errorCode << std::endl;
}
```

2. `Class Types`: This is the most common and recommended approach. The C++ Standard Library provides a hierarchy of exception classes derived from std::exception (defined in exception). Common standard exceptions (often found in stdexcept) include:
	- `std::exception`: Base class for standard exceptions. Catching const std::exception& can handle many library errors.
	- `std::runtime_error`: Errors detectable only at runtime (e.g., std::overflow_error, std::underflow_error).
	- std::logic_error: Errors representing flaws in internal program logic 
			(e.g., std::invalid_argument, std::out_of_range, std::length_error).
	- std::bad_alloc: Thrown by new on memory allocation failure.
	- std::bad_cast: Thrown by dynamic_cast on failure to cast a reference.

You can (and should) define your own custom exception classes, often inheriting from std::exception or one of its derivatives.

```cpp
#include <stdexcept> // Required for std::runtime_error
#include <iostream>  // Required for cerr

try {
    // Some operation that might fail
    if (/* condition */ true) { // Simplified condition
        throw std::runtime_error("Something went wrong during runtime!");
    }
} catch (const std::runtime_error& e) { // Catch standard runtime errors by const reference
    std::cerr << "Caught a runtime error: " << e.what() << std::endl; // .what() gives error description
} catch (const std::exception& e) { // Catch any other standard exception
    std::cerr << "Caught a standard exception: " << e.what() << std::endl;
}
```

3. `Pointers`: You can throw and catch pointers, though this is less common for error handling than throwing objects or primitive types.

4. `Catch-All (catch (...))`: The ellipsis ... acts as a wildcard, catching any type of exception. This is useful as a last resort handler to ensure an exception doesn't go unhandled, perhaps for logging or performing essential cleanup. However, it provides no information about the type or value of the exception caught.

# **Related Concepts:**

## Relationship with try and throw

[[catch]] is intrinsically linked to [[try]]. It only functions immediately following a try block (or another catch for the same try).

It exists to handle exceptions signaled by the [[throw]] keyword from within the associated try block.
# **Examples:**

### Example Context Revisited

```cpp
try {
    int age = 15;
    if (age >= 18) {
        cout << "Access granted - you are old enough.";
    } else {
        throw (age); // Throws an exception of type int
    }
}
catch (int myNum) { // This block is specifically designed to catch 'int' exceptions
    // Code here executes because an int was thrown and this handler matches.
    cout << "Access denied - You must be at least 18 years old.\n";
    cout << "Age is: " << myNum; // 'myNum' gets the value of the thrown 'age' (15)
}
```

In this specific case, the catch block is tailored to handle int exceptions. The parameter myNum receives the value 15 that was thrown. If the throw statement had thrown a different type (e.g., throw std::runtime_error("Too young");), this catch (int myNum) block would not have executed. Another appropriate catch block (e.g., catch(const std::runtime_error& e)) would be needed.