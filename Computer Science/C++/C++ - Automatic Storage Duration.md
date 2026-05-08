---
memory: to_finish
tags:
 - learned
language:
 - C++
review-date: ""
last-reviewed: 2025-08-14
scheda: done
cssclasses:
visit-count: 4
confidence-level: 2
consecutive-correct: 3
last-struggle-date: 2025-07-05

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
Automatic storage duration in C++ addresses the need for efficient and predictable memory management ==for temporary data within functions and blocks of code==. It solves the problem of how to allocate and deallocate memory for variables that are only needed for a limited time, specifically while a function is executing or a block of code is active.

Its primary application is for local variables within functions, function parameters, and variables declared within block scopes (e.g., inside `if` statements or `for` loops).

It's important in computer science and C++ because it provides:

- **Efficiency:** ==Memory for automatic variables is typically allocated on the program's call stack==, which is an extremely fast allocation and deallocation mechanism. This avoids the overhead associated with dynamic memory allocation (e.g., using `new` and `delete`).
- **Predictability:** The lifetime of automatic variables is directly tied to their scope. They are created when their scope is entered and destroyed when their scope is exited. This deterministic behavior makes it easier to reason about program state and avoids memory leaks or dangling pointers that can arise from manual memory management.
- **Encapsulation:** Automatic variables are local to their scope, meaning they cannot be accessed from outside that scope. This promotes good programming practices by limiting the visibility and potential side effects of variables.

# **Core Explanation:**

---
**Definition:** Automatic storage duration refers to a<mark style="background:

# FF5582A6;"> type of storage duration in C++ where an object's lifetime is tied to its scope</mark>. Variables with automatic storage duration are typically allocated on the program's call stack.

**Key Characteristics:**

- **Scope-bound Lifetime:** An object with automatic storage duration is created when its enclosing scope is entered and destroyed when that scope is exited. This "scope" can be a function body, a block within a function (e.g., an `if` block, a `for` loop body), or even a temporary object created in an expression.
- **Stack Allocation:** The memory for automatic variables is usually allocated on the program's **call stack**. The stack is a LIFO (Last-In, First-Out) data structure that grows and shrinks as functions are called and return.
- **Automatic Deallocation:** Memory for automatic variables is automatically reclaimed when they go out of scope. You do not need to explicitly deallocate them (e.g., with `delete`). This prevents memory leaks.
- **Default for Local Variables:** Local variables (variables declared inside a function or block) that are not explicitly given another storage duration specifier (like `static` or `thread_local`) have automatic storage duration by default.
- **No Initial Value Guarantee:** Unless explicitly initialized, automatic variables hold indeterminate values (garbage). Accessing an uninitialized automatic variable leads to undefined behavior.

**How it Works:**

When a function is called,<mark style="background:

# BBFABBA6;"> a new "stack frame" is pushed onto the call stack.</mark> This stack frame contains <mark style="background:

# BBFABBA6;">space for the function's parameters, its local automatic variables, and other administrative information. As the function executes, its local automatic variables are created within this stack frame.</mark>

When the <mark style="background:

# FFB8EBA6;">function finishes execution</mark> (either by returning or throwing an exception),<mark style="background:

# FFB8EBA6;"> its stack frame is popped off the call stack. This action automatically deallocates all the memory associated with the function's parameters and local automatic variables.</mark>

The `auto` [[C++ - Auto Keyword]] keyword, <mark style="background:

# D2B3FFA6;">when used as a storage class specifier (prior to C++11), explicitly declared a variable with automatic storage duration</mark>. However, its use for this purpose is now largely superseded by its role in type deduction (since C++11). Still, understanding that most local variables implicitly have `auto` storage duration is fundamental.

# **Related Concepts:**

---
- **Scope (Block Scope, Function Scope):** Scope defines the region of a program where a declared name is valid and accessible. Automatic storage duration is directly tied to scope: variables with automatic storage duration exist only within their defined scope. When the program execution leaves that scope, the automatic variable is destroyed.

- **[[Call Stack]]:** The call stack is the fundamental data structure that supports automatic storage duration. It's a region of memory used to manage function calls. Each time a function is called, a new "stack frame" is pushed onto the stack, containing local variables and function parameters. When a function returns, its stack frame is popped, effectively deallocating the memory for its automatic variables.

- **Static Storage Duration:** In contrast to automatic storage duration, objects with static storage duration exist for the entire lifetime of the program. They are allocated at program startup and deallocated at program termination. This applies to global variables, static local variables, and static member variables. They are typically stored in the data segment or BSS segment, not the stack.

- **Dynamic Storage Duration (Free Store/Heap):** This refers to memory allocated and deallocated explicitly by the programmer using `new` and `delete` (or `malloc` and `free` in C-style programming). Objects with dynamic storage duration persist until explicitly deallocated. They are allocated on the "heap" (also known as the free store), a much larger pool of memory than the stack. Unlike automatic variables, managing dynamic memory requires careful attention to avoid memory leaks or double deallocation.

- **Thread Local Storage Duration:** Introduced in C++11, this applies to variables that have a unique instance per thread. Their lifetime is tied to the lifetime of the thread they belong to. They are somewhat similar to static variables but are localized to each thread.

- **RAII (Resource Acquisition Is Initialization):** This is a C++ programming idiom that leverages automatic storage duration to manage resources. Resources (like file handles, network connections, or dynamically allocated memory) are acquired in the constructor of an object and released in its destructor. By associating resource management with the lifetime of an automatic object, RAII ensures that resources are automatically released when the object goes out of scope, preventing leaks. Smart pointers (`std::unique_ptr`, `std::shared_ptr`) are prime examples of RAII in action, managing dynamically allocated memory using automatic objects.

# **Examples:**

---
```cpp

# include <iostream>

# include <string>

// Function demonstrating automatic storage duration for parameters and local variables
void demonstrateAutomaticStorage(int a, double b) {
 // 'a' and 'b' are function parameters. They have automatic storage duration.
 // They are created when demonstrateAutomaticStorage is called and destroyed when it returns.
 std::cout << "Inside demonstrateAutomaticStorage function." << std::endl;
 std::cout << "Parameter a: " << a << std::endl;
 std::cout << "Parameter b: " << b << std::endl;

 // 'local_variable' has automatic storage duration.
 // It's created when this function is entered.
 int local_variable = 100;
 std::cout << "local_variable: " << local_variable << std::endl;

 // 'another_string' also has automatic storage duration.
 std::string another_string = "Hello from inside the function!";
 std::cout << "another_string: " << another_string << std::endl;

 { // Start of a new block scope
 // 'block_scoped_variable' has automatic storage duration, local to this inner block.
 // It's created when this block is entered.
 bool block_scoped_variable = true;
 std::cout << "Inside inner block. block_scoped_variable: " << block_scoped_variable << std::endl;
 } // End of inner block scope. 'block_scoped_variable' is destroyed here.
 // std::cout << block_scoped_variable << std::endl; // ERROR: block_scoped_variable is out of scope

 std::cout << "Exiting demonstrateAutomaticStorage function." << std::endl;
 // 'local_variable', 'another_string', 'a', and 'b' are destroyed here.
}

int main {
 std::cout << "Starting main function." << std::endl;

 // 'main_variable' has automatic storage duration, local to main.
 int main_variable = 50;
 std::cout << "main_variable in main: " << main_variable << std::endl;

 // Call the function, passing values.
 // The values 10 and 20 are passed by value, creating copies in 'a' and 'b'
 // within the function's stack frame.
 demonstrateAutomaticStorage(10, 20.5);

 std::cout << "Back in main function." << std::endl;
 // 'main_variable' is still in scope and accessible.
 std::cout << "main_variable after function call: " << main_variable << std::endl;

 { // Another block scope in main
 std::string temporary_message = "This message is temporary.";
 std::cout << temporary_message << std::endl;
 } // 'temporary_message' is destroyed here.

 std::cout << "Exiting main function." << std::endl;
 // 'main_variable' is destroyed here as main's scope ends.
 return 0;
}
```

# **Flashcards:**

---
What is automatic storage duration in C++?;;A type of storage duration where an object's lifetime is tied to its scope, typically allocated on the call stack.

When are objects with automatic storage duration created and destroyed?;;They are created when their enclosing scope is entered and destroyed when that scope is exited.

What is the primary benefit of automatic storage duration?;;Efficient and predictable memory management for temporary variables, avoiding manual deallocation and common memory errors.