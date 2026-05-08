---
memory: to_finish
tags:
  - learned
language:
  - C++
review-date: ""
last-reviewed: 2025-09-01
scheda: done
visit-count: 2
confidence-level: 1.5
consecutive-correct: 1
last-struggle-date: 2025-07-01
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

The **Call Stack** (often just "the stack") is a crucial runtime mechanism used by C++ (and many other programming languages) to manage active function calls. ==It's a region of memory that operates on a **LIFO (Last-In, First-Out)** principle.==

Its primary purposes are:

1. **Tracking Active Functions**: To keep track of the point to which each active function should return control when it finishes executing.
2. **Storing Local Variables**: To allocate memory for local variables and parameters of functions.
3. **Passing Arguments**: To help in passing arguments to functions.
4. **Storing Return Values**: Sometimes, space for a function's return value is also managed on the stack (especially for small values).

<mark style="background: #BBFABBA6;">When a function is called, a new **stack frame** (also known as an activation record) is "pushed" onto the top of the call stack. When a function finishes and returns, its stack frame is "popped" off the stack, and control returns to the address specified in the now-topmost frame.</mark>

---

## How it Works in C++:

In C++, each time a function is invoked:

1. **Stack Frame Creation**: A new stack frame is allocated on the call stack for that function call. This frame typically contains:
    
    - **Return Address**: The memory address in the caller function where execution should resume after the current function completes.
    - **Parameters (Arguments)**: Values passed to the function by the caller.
    - **Local Variables**: Variables declared inside the function. Their memory is allocated within this frame.
    - **Saved Registers**: Sometimes, the state of CPU registers from the caller is saved so they can be restored when the function returns.
2. **Push Operation**: The newly created stack frame is pushed onto the top of the stack. Execution then jumps to the beginning of the called function.
    
3. **Execution**: The function executes. It can access its parameters and local variables stored in its own stack frame. It might also call other functions, which would lead to more stack frames being pushed onto the stack.
    
4. **Pop Operation**: When the function finishes (e.g., via a `return` statement or by reaching the end of the function):
    
    - Its local variables go out of scope and the memory they occupied is effectively deallocated (the stack pointer moves).
    - The return value (if any) is typically placed in a CPU register or a specific memory location accessible by the caller.
    - The stack frame is popped off the call stack.
    - Execution jumps back to the **return address** stored in the (now previous) calling function's stack frame.
---
# **Related Concepts:**

- **[[Recursion]]**: When a function calls itself, directly or indirectly. Each recursive call creates a new stack frame. This can lead to rapid growth of the call stack.
- **[[Stack Overflow]]**: A runtime error that occurs when the call stack exceeds its allocated size. This typically happens due to excessively deep recursion (e.g., infinite recursion or recursion with a very large number of calls without a base case being met quickly enough) or if functions allocate very large local variables on the stack.
- **[[C++ - Automatic Storage Duration]]**: Local variables in C++ (those declared within a function without `static`) have automatic storage duration. They are created when their stack frame is pushed and destroyed when it's popped. This is why their memory is managed on the call stack.
- **[[C++ - Debugging Tools]]**: Understanding the call stack is vital for debugging. Debuggers allow you to inspect the call stack to see the sequence of function calls that led to the current point of execution, along with the local variables in each frame.
---
# **Examples:**

**Example 1: Simple Nested Calls**

```c++
#include <iostream>

void functionB(int b_param) {
    int b_local = 20;
    std::cout << "Inside functionB. Param: " << b_param << ", Local: " << b_local << std::endl;
    // When functionB returns, its frame is popped.
}

void functionA(int a_param) {
    int a_local = 10;
    std::cout << "Inside functionA. Param: " << a_param << ", Local: " << a_local << std::endl;
    functionB(a_param + 5); // Call functionB
    std::cout << "Back in functionA." << std::endl;
    // When functionA returns, its frame is popped.
}

int main() {
    std::cout << "Inside main." << std::endl;
    functionA(5); // Call functionA
    std::cout << "Back in main. Program ends." << std::endl;
    // main's frame is popped.
    return 0;
}
```

**Call Stack Order (Conceptual):**

1. `main()` is called:
    - Stack: `[main frame]`
2. `main()` calls `functionA(5)`:
    - Stack: `[main frame] <- [functionA frame (a_param=5, a_local=10)]`
3. `functionA()` calls `functionB(10)`:
    - Stack: `[main frame] <- [functionA frame] <- [functionB frame (b_param=10, b_local=20)]`
4. `functionB()` returns:
    - Stack: `[main frame] <- [functionA frame]` (Control back to `functionA`)
5. `functionA()` returns:
    - Stack: `[main frame]` (Control back to `main`)
6. `main()` returns:
    - Stack: `[]` (Empty)

**Example 2: Recursion (Factorial)**

```c++
#include <iostream>

long factorial(int n) {
    if (n <= 1) { // Base case
        std::cout << "Base case: factorial(" << n << ")" << std::endl;
        return 1;
    } else { // Recursive step
        std::cout << "Recursive call: factorial(" << n << ") calls factorial(" << n-1 << ")" << std::endl;
        return n * factorial(n - 1);
    }
}

int main() {
    int num = 3;
    std::cout << "Calculating factorial of " << num << std::endl;
    long result = factorial(num);
    std::cout << "Factorial of " << num << " is " << result << std::endl;
    return 0;
}
```

When `factorial(3)` is called, it pushes frames for `factorial(3)`, then `factorial(2)`, then `factorial(1)`. `factorial(1)` hits the base case and returns, then `factorial(2)` returns, and finally `factorial(3)` returns.

---

## Is it a topic in other languages? If yes, explain that.

**Yes, absolutely.** The concept of a call stack is fundamental to how most modern imperative, functional, and object-oriented programming languages manage function/method execution.

**General Explanation for Other Languages:**

- **Ubiquitous Concept**: Languages like Java, Python, C#, [[JavaScript]], Ruby, Go, Swift, etc., all use a call stack (or an equivalent mechanism).
- **Core Functionality is the Same**: The LIFO principle, the storage of return addresses, local variables, and parameters within stack frames, and the push/pop operations upon function call/return are common across these languages.
- **Implementation Details Vary**:
    - **Memory Management**: While C++ requires manual memory management for heap objects, languages like Java, Python, and C# have automatic garbage collection. The call stack itself is still managed similarly for function calls and local variables (which are typically stack-allocated or references to heap-allocated objects).
    - **Error Handling**: How stack overflows are reported (e.g., `StackOverflowError` in Java, `RecursionError` in Python) can differ.
    - **Virtual Machines/Interpreters**: Languages running on virtual machines (like Java's JVM or Python's interpreter) have their own defined call stack management within that environment.
    - **Language Features**: Specific language features (e.g., closures in JavaScript, coroutines/goroutines) might have more complex interactions with or extensions to the basic call stack model, sometimes involving heap-allocated activation records or multiple stacks.

**In summary, while the exact low-level implementation details and the surrounding ecosystem (like garbage collection) may differ, the call stack as a mechanism for managing the flow of control and local data for subroutines is a nearly universal concept in structured programming.** Understanding it helps in debugging, performance analysis, and grasping how programs execute, regardless of the specific language.