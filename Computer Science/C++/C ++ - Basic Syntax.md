---
memory: to_finish
tags:
  - mastered
language:
  - C++
review-date: ""
last-reviewed: 2025-08-22
scheda: done
visit-count: 1
confidence-level: 1.5
consecutive-correct: 1
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


C++ is an object-oriented programming language that is an extension of the C programming language. It maintains a lot of the same syntax and structure as C, but with additional features and capabilities. Let's explore the basic syntax elements you'll encounter when writing C++ code.

[[C++ Advanced Syntax]]
### Program Structure
A C++ program consists of one or more files, each containing one or more functions. The execution of a C++ program typically starts from the `main()` function, which is the entry point of the program.

The basic structure of a C++ program looks like this:

```cpp
#include <iostream>

int main() {
    // Your code goes here
    return 0;
}
```

The `#include <iostream>` line is a preprocessor directive that includes the standard input/output stream library, which allows you to perform input and output operations in your program.

The `main()` function is where the execution of your program begins. It has a return type of `int`, which means it should return an integer value, typically `0` to indicate successful program execution.

Within the `main()` function, you'll write your actual C++ code, which can include variable declarations, statements, and function calls.
### Variables
In C++, you declare variables using a data type followed by the variable name. The data type determines the type of value the variable can hold. Common data types include:

- `int`: Stores integer values (e.g., `42`, `-10`, `0`)
- `float`: Stores floating-point numbers (e.g., `3.14`, `-2.5`, `0.0`)
- `double`: Stores larger floating-point numbers with more precision
- `char`: Stores a single character (e.g., `'a'`, `'1'`, `' '`)
- `bool`: Stores a boolean value (`true` or `false`)

Here's an example of variable declarations:

```cpp
int age = 25;
float pi = 3.14159;
char grade = 'A';
bool isStudent = true;
```

You can also declare multiple variables of the same type in a single line, separated by commas:

```cpp
int x, y, z;
```

Variables can be initialized when they are declared, as shown in the first example, or they can be left uninitialized, in which case they will have a default value (usually 0 or `false`).

### Operators
C++ provides a variety of operators that you can use to perform various operations on variables and values. Some common operators include:

- Arithmetic operators: `+`, `-`, `*`, `/`, `%` (modulus)
- Assignment operators: `(single equal sign)`, `+=`, `-=`, `*=`, `/=`
- Comparison operators: `(double equal sign)`, `!=`, `<`, `>`, `<=`, `>=`
- Logical operators: `&&` (and), `||` (or), `!` (not)
- Bitwise operators: `&` (and), `|` (or), `^` (xor), `~` (not), `<<` (left shift), `>>` (right shift)

Here's an example of using some of these operators:

```cpp
int a = 10;
int b = 4;
int c = a + b;  // c = 14
bool isGreater = a > b;  // isGreater = true
```

In this example, we perform addition (`+`) to get the sum of `a` and `b`, and we use the greater-than (`>`) operator to compare `a` and `b` and store the result in the `isGreater` variable.

### [[C++ Control Flow]]
C++ provides several control flow statements that allow you to control the execution of your program based on certain conditions or loops. The most common ones are:

1. **If-Else Statements**:
   ```cpp
   if (condition) {
       // Code to be executed if the condition is true
   } else {
       // Code to be executed if the condition is false
   }
   ```
   The `if-else` statement allows you to execute different blocks of code based on a given condition.

2. **Switch Statements**:
   ```cpp
   switch (expression) {
       case value1:
           // Code to be executed if expression matches value1
           break;
       case value2:
           // Code to be executed if expression matches value2
           break;
       ...
       default:
           // Code to be executed if expression doesn't match any case
   }
   ```
   The `switch` statement provides a way to execute different blocks of code based on the value of a given expression.

3. **Loops**:
   - `for` loop:
     ```cpp
     for (initialization; condition; increment/decrement) {
         // Code to be executed
     }
     ```
   - `while` loop:
     ```cpp
     while (condition) {
         // Code to be executed
     }
     ```
   - `do-while` loop:
     ```cpp
     do {
         // Code to be executed
     } while (condition);
     ```
   Loops allow you to repeatedly execute a block of code until a certain condition is met.

Control flow statements are essential for implementing logical decision-making and repetitive tasks in your C++ programs.
### Functions
>Functions in C++ are blocks of code that perform a specific task and can be called from other parts of the program. The basic syntax for defining a function is:

```cpp
return_type function_name(parameter_list) {
    // Function body
    return value;
}
```

>- `return_type`: The data type of the value the function will return, or `void` if the function doesn't return anything.
>- `function_name`: The name you give to the function.
>- `parameter_list`: A comma-separated list of parameters the function takes, or an empty set of parentheses if the function takes no parameters.
>- `function body`: The code that will be executed when the function is called.
>- `return value`: The value the function will return, if any.

Here's an example of a simple function:

```cpp
int add(int a, int b) {
    int sum = a + b;
    return sum;
}
```

This function takes two integer parameters, `a` and `b`, adds them together, and returns the result as an integer.

Functions are a fundamental building block of C++ programs, allowing you to organize your code, reuse functionality, and improve readability and maintainability.


