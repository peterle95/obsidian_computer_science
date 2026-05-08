---
memory: to_finish
tags:
 - learned
language:
 - Core Concepts
review-date: ""
last-reviewed: 2025-08-15
scheda: done
visit-count: 2
confidence-level: 1
consecutive-correct: 1
last-struggle-date: 2025-08-05

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
Stack overflow addresses the fundamental problem of preventing ==infinite or excessive memory consumption in the call stack region of program memory==. It serves as a protective mechanism that detects when a program is consuming too much stack space, typically due to r==unaway recursion or excessive local variable allocation==. This concept is crucial in computer science because it prevents programs from consuming all available system memory, which could crash the entire system or cause other applications to fail. Understanding stack overflow is essential for writing robust code, debugging recursive algorithms, and managing memory efficiently in programming languages that use stack-based memory allocation for function calls and local variables.

# **Core Explanation:**

---
<mark style="background:

# FF5582A6;">Stack overflow is a runtime error that occurs when a program's call stack exceeds its allocated memory limit</mark>. The call stack is a region of memory used to store information about active function calls, including local variables, function parameters, return addresses, and execution context.

Key characteristics of stack overflow include:

**Causes**: The <mark style="background:

# BBFABBA6;">most common cause is infinite or deeply nested recursion where functions call themselves or each other without proper termination conditions</mark>. Other causes include allocating very large local variables or arrays on the stack, or having extremely deep function call chains.

**Memory Structure**: The stack grows and shrinks as functions are called and return. Each function call adds a new stack frame containing local variables and metadata. When functions return, their frames are removed (popped) from the stack.

**Detection Mechanism**: Operating systems and runtime environments set stack size limits (typically 1-8 MB). When a program attempts to use more stack space than allocated, the system throws a stack overflow exception or terminates the program.

**Stack vs Heap**: Unlike heap memory which is used for dynamic allocation and can grow as needed (limited by available RAM), stack memory has fixed size limits and is managed automatically by the program's execution model.

**Recovery**: Stack overflow is typically unrecoverable at runtime because it represents a fundamental programming error rather than a temporary resource shortage. Programs must be fixed at the source code level.

The error manifests differently across programming languages but generally results in program termination with an error message indicating stack space exhaustion.

# **Related Concepts:**

---
**Call Stack**: The underlying data structure where stack overflow occurs, containing stack frames for each active function call with local variables, parameters, and return addresses.

**Recursion**: The primary programming technique that leads to stack overflow when base cases are missing or incorrect, causing functions to call themselves indefinitely.

**Tail Recursion Optimization**: A compiler optimization that prevents stack overflow in recursive functions by reusing the current stack frame instead of creating new ones for tail-recursive calls.

**Stack Frame**: Individual memory units on the call stack containing a function's local variables, parameters, and execution context, which accumulates during deep recursion.

**Memory Management**: The broader concept of how programs allocate and deallocate memory, where stack overflow represents failure in automatic stack memory management.

**Heap Overflow**: A different type of memory error occurring in dynamically allocated memory, contrasting with stack overflow which occurs in automatically managed function call memory.

**Stack Size Limits**: Operating system and runtime environment configurations that determine when stack overflow errors are triggered, varying between platforms and languages.

**Exception Handling**: The mechanism by which programs detect and respond to stack overflow, though recovery is typically impossible once the error occurs.

# **Examples:**

---
```python

# Python Stack Overflow Examples

# Python typically has a default recursion limit of ~1000 calls

import sys

def infinite_recursion(n):
 """
 This function will cause stack overflow due to infinite recursion
 Each call creates a new stack frame with parameter 'n'
 Stack frames accumulate because there's no base case to stop recursion
 """
 print(f"Call number: {n}")

# This line executes before stack overflow
 return infinite_recursion(n + 1)

# Recursive call without termination

def factorial_bad(n):
 """
 Poorly implemented factorial that can cause stack overflow
 for large values of n due to deep recursion
 """
 if n <= 1:
 return 1

# Each recursive call adds new frame to call stack

# For factorial(10000), this creates 10000 stack frames
 return n * factorial_bad(n - 1)

def factorial_good(n):
 """
 Iterative implementation that avoids stack overflow
 Uses constant stack space regardless of input size
 """
 result = 1

# Loop uses same stack frame throughout execution
 for i in range(1, n + 1):
 result *= i

# No recursive calls, no stack buildup
 return result

def demonstrate_stack_overflow:
 """
 Demonstrate different scenarios that cause stack overflow
 """

# Check current recursion limit
 print(f"Python recursion limit: {sys.getrecursionlimit}")

 try:

# This will cause RecursionError (Python's stack overflow)
 infinite_recursion(1)
 except RecursionError as e:
 print(f"Stack overflow caught: {e}")

 try:

# Large factorial will also cause stack overflow
 result = factorial_bad(5000)
 except RecursionError as e:
 print(f"Factorial recursion error: {e}")

# Safe iterative version works fine
 result = factorial_good(5000)
 print(f"Iterative factorial succeeded, result length: {len(str(result))}")

# Example of proper recursive function with base case
def fibonacci_safe(n, memo={}):
 """
 Fibonacci with memoization to prevent redundant recursive calls
 Base cases prevent infinite recursion
 Memoization reduces call depth significantly
 """
 if n in memo:

# Check if already computed
 return memo[n]

 if n <= 1:

# Base case prevents infinite recursion
 return n

# Store result to avoid recomputation
 memo[n] = fibonacci_safe(n-1, memo) + fibonacci_safe(n-2, memo)
 return memo[n]

if __name__ == "__main__":
 demonstrate_stack_overflow
```

```java
// Java Stack Overflow Examples
// Java throws StackOverflowError when stack space is exhausted

public class StackOverflowDemo {

 // Static counter to track recursion depth
 private static int callCount = 0;

 /**
 * Infinite recursion method that will cause StackOverflowError
 * Each method call consumes stack space for local variables and return address
 */
 public static void infiniteRecursion {
 callCount++; // Local operation before stack overflow
 System.out.println("Call: " + callCount);
 // No base case - this will continue until stack overflow
 infiniteRecursion; // Recursive call creates new stack frame
 }

 /**
 * Factorial implementation that can cause stack overflow
 * for large inputs due to deep recursion
 */
 public static long factorialRecursive(int n) {
 // Base case prevents infinite recursion
 if (n <= 1) {
 return 1;
 }
 // Each recursive call adds new frame to call stack
 // Stack depth equals the value of n
 return n * factorialRecursive(n - 1);
 }

 /**
 * Iterative factorial that avoids stack overflow
 * Uses constant stack space regardless of input size
 */
 public static long factorialIterative(int n) {
 long result = 1;
 // Loop reuses same stack frame
 for (int i = 1; i <= n; i++) {
 result *= i; // No method calls, no stack growth
 }
 return result;
 }

 /**
 * Method that creates large local arrays on stack
 * Can cause stack overflow if arrays are too large
 */
 public static void largeLocalArrays {
 // Large arrays allocated on stack (in some JVM implementations)
 // Stack space is limited compared to heap space
 int largeArray1 = new int; // Stack allocation
 int largeArray2 = new int; // Additional stack usage
 int largeArray3 = new int; // May exceed stack limit

 // Recursive call with large local variables
 if (largeArray1.length > 0) {
 largeLocalArrays; // Adds more stack frames with large locals
 }
 }

 /**
 * Tail recursive method - some JVMs can optimize this
 * But Java doesn't guarantee tail call optimization
 */
 public static long factorialTailRecursive(int n, long accumulator) {
 if (n <= 1) {
 return accumulator; // Base case with accumulated result
 }
 // Tail recursive call - last operation in method
 // Could be optimized to reuse current stack frame
 return factorialTailRecursive(n - 1, n * accumulator);
 }

 public static void main(String args) {
 System.out.println("=== Java Stack Overflow Demonstration ===");

 // Example 1: Infinite recursion
 try {
 infiniteRecursion;
 } catch (StackOverflowError e) {
 System.out.println("StackOverflowError caught after " + callCount + " calls");
 System.out.println("Error message: " + e.getMessage);
 }

 // Example 2: Deep recursion with factorial
 try {
 long result = factorialRecursive(10000); // Too deep for most JVMs
 System.out.println("Recursive factorial result: " + result);
 } catch (StackOverflowError e) {
 System.out.println("Factorial recursion caused stack overflow");
 }

 // Example 3: Safe iterative version
 try {
 long result = factorialIterative(20); // Works fine
 System.out.println("Iterative factorial(20): " + result);
 } catch (Exception e) {
 System.out.println("Iterative factorial failed: " + e.getMessage);
 }

 // Example 4: Large local variables
 try {
 largeLocalArrays;
 } catch (StackOverflowError e) {
 System.out.println("Large local arrays caused stack overflow");
 }
 }
}
```

```c
/*
C Stack Overflow Examples
C doesn't have built-in stack overflow detection
Program may crash with segmentation fault when stack limit exceeded
*/

# include <stdio.h>

# include <stdlib.h>

// Global counter to track recursion depth before crash
int recursion_depth = 0;

/**
 * Infinite recursion function
 * Each call pushes new frame onto stack with local variables
 * C doesn't check stack bounds - will eventually crash
 */
void infinite_recursion {
 int local_var = 42; // Local variable consumes stack space
 recursion_depth++; // Increment counter

 // Print periodically to show progress before crash
 if (recursion_depth % 10000 == 0) {
 printf("Recursion depth: %d\n", recursion_depth);
 }

 // Recursive call - no base case, will eventually exhaust stack
 infinite_recursion;
}

/**
 * Function that allocates large local arrays
 * Large stack allocations can quickly exhaust stack space
 */
void large_stack_allocation {
 // Large array allocated on stack
 // Stack space is typically 1-8MB, this may exceed it
 char large_array[1024 * 1024]; // 1MB array on stack

 // Initialize array to prevent compiler optimization
 for (int i = 0; i < sizeof(large_array); i++) {
 large_array[i] = i % 256;
 }

 printf("Allocated %zu bytes on stack\n", sizeof(large_array));

 // Recursive call with large local allocation
 large_stack_allocation; // Each call adds 1MB to stack
}

/**
 * Recursive factorial that can cause stack overflow
 * Stack depth equals input value n
 */
long factorial_recursive(int n) {
 // Base case prevents infinite recursion
 if (n <= 1) {
 return 1;
 }

 // Each recursive call consumes stack space
 // For large n, this will exhaust available stack
 return n * factorial_recursive(n - 1);
}

/**
 * Iterative factorial that uses constant stack space
 * Safe alternative to recursive version
 */
long factorial_iterative(int n) {
 long result = 1;

 // Loop uses same stack frame throughout execution
 for (int i = 1; i <= n; i++) {
 result *= i; // No function calls, constant stack usage
 }

 return result;
}

/**
 * Demonstration of tail recursion in C
 * C compilers may optimize tail calls, but not guaranteed
 */
long factorial_tail_recursive(int n, long accumulator) {
 if (n <= 1) {
 return accumulator;
 }

 // Tail recursive call - last operation in function
 // Some compilers optimize this to use same stack frame
 return factorial_tail_recursive(n - 1, n * accumulator);
}

int main {
 printf("=== C Stack Overflow Demonstration ===\n");

 // Example 1: Test iterative factorial (safe)
 printf("Factorial(20) iterative: %ld\n", factorial_iterative(20));

 // Example 2: Test recursive factorial with reasonable input
 printf("Factorial(10) recursive: %ld\n", factorial_recursive(10));

 // WARNING: The following examples will likely crash the program
 // Uncomment only one at a time for testing

 /*
 // Example 3: Large stack allocation
 // This will likely cause segmentation fault
 printf("Testing large stack allocation...\n");
 large_stack_allocation;
 */

 /*
 // Example 4: Infinite recursion
 // This will eventually crash with segmentation fault
 printf("Testing infinite recursion...\n");
 infinite_recursion;
 printf("This line will never execute\n");
 */

 /*
 // Example 5: Deep recursion with large input
 // This will likely cause stack overflow
 printf("Testing deep recursion...\n");
 long result = factorial_recursive(100000);
 printf("Deep recursion result: %ld\n", result);
 */

 return 0;
}
```

```javascript
// JavaScript Stack Overflow Examples
// JavaScript throws "RangeError: Maximum call stack size exceeded"

// Track recursion depth
let callCount = 0;

/**
 * Infinite recursion function
 * JavaScript engines have call stack limits (typically 10,000-100,000 calls)
 */
function infiniteRecursion(n) {
 callCount++;

 // Log progress periodically
 if (callCount % 1000 === 0) {
 console.log(`Call count: ${callCount}`);
 }

 // No base case - will continue until stack overflow
 return infiniteRecursion(n + 1);
}

/**
 * Recursive factorial that can cause stack overflow
 * Each call creates new execution context on call stack
 */
function factorialRecursive(n) {
 // Base case prevents infinite recursion
 if (n <= 1) return 1;

 // Recursive call adds new frame to call stack
 // Stack depth equals the value of n
 return n * factorialRecursive(n - 1);
}

/**
 * Iterative factorial using constant stack space
 * Safe alternative that won't cause stack overflow
 */
function factorialIterative(n) {
 let result = 1;

 // Loop reuses same execution context
 for (let i = 1; i <= n; i++) {
 result *= i; // No function calls, no stack growth
 }

 return result;
}

/**
 * Mutual recursion example - functions calling each other
 * Can cause stack overflow if no proper termination
 */
function isEven(n) {
 if (n === 0) return true; // Base case
 return isOdd(n - 1); // Call to other function
}

function isOdd(n) {
 if (n === 0) return false; // Base case
 return isEven(n - 1); // Call back to first function
}

/**
 * Demonstrate proper recursion with memoization
 * Prevents stack overflow by avoiding redundant calculations
 */
function fibonacciMemoized(n, memo = {}) {
 // Check if result already computed
 if (n in memo) return memo[n];

 // Base cases
 if (n <= 1) return n;

 // Store result to avoid recomputation
 memo[n] = fibonacciMemoized(n - 1, memo) + fibonacciMemoized(n - 2, memo);
 return memo[n];
}

/**
 * Convert recursion to iteration using explicit stack
 * Simulates recursion without using call stack
 */
function factorialIterativeStack(n) {
 // Use array as explicit stack instead of call stack
 const stack = ;
 let result = 1;

 // Push all values onto our stack
 for (let i = n; i > 1; i--) {
 stack.push(i);
 }

 // Process stack without recursive calls
 while (stack.length > 0) {
 result *= stack.pop; // No function calls, just array operations
 }

 return result;
}

/**
 * Demonstration function with error handling
 */
function demonstrateStackOverflow {
 console.log("=== JavaScript Stack Overflow Demonstration ===");

 // Example 1: Safe iterative factorial
 try {
 const result = factorialIterative(20);
 console.log(`Iterative factorial(20): ${result}`);
 } catch (error) {
 console.log(`Iterative factorial error: ${error.message}`);
 }

 // Example 2: Recursive factorial with reasonable input
 try {
 const result = factorialRecursive(10);
 console.log(`Recursive factorial(10): ${result}`);
 } catch (error) {
 console.log(`Recursive factorial error: ${error.message}`);
 }

 // Example 3: Mutual recursion with reasonable input
 try {
 console.log(`isEven(100): ${isEven(100)}`);
 console.log(`isOdd(101): ${isOdd(101)}`);
 } catch (error) {
 console.log(`Mutual recursion error: ${error.message}`);
 }

 // Example 4: Memoized fibonacci (safe for large inputs)
 try {
 const result = fibonacciMemoized(100);
 console.log(`Fibonacci(100) with memoization: ${result}`);
 } catch (error) {
 console.log(`Memoized fibonacci error: ${error.message}`);
 }

 // Example 5: Test infinite recursion (will cause stack overflow)
 try {
 callCount = 0; // Reset counter
 infiniteRecursion(1);
 } catch (error) {
 console.log(`Stack overflow after ${callCount} calls`);
 console.log(`Error type: ${error.name}`);
 console.log(`Error message: ${error.message}`);
 }

 // Example 6: Deep recursion that causes stack overflow
 try {
 const result = factorialRecursive(100000); // Too deep for most engines
 console.log(`Deep recursion result: ${result}`);
 } catch (error) {
 console.log(`Deep recursion caused: ${error.name} - ${error.message}`);
 }
}

// Run demonstration
demonstrateStackOverflow;
```

# **Flashcards:**

---
What is stack overflow and what are its most common causes?;; Stack overflow is a runtime error that occurs when a program's call stack exceeds its allocated memory limit. The most common causes are infinite or deeply nested recursion without proper base cases, allocating very large local variables on the stack, and extremely deep function call chains that exhaust available stack space.

Which programming languages are susceptible to stack overflow and how do they handle it?;; Most programming languages with function call stacks are susceptible, including C/C++ (segmentation fault), Java (StackOverflowError), Python (RecursionError), JavaScript (RangeError), C

# (StackOverflowException), and others. Languages differ in detection mechanisms and stack size limits, but all use stack-based memory for function calls and local variables.

What are effective strategies to prevent or avoid stack overflow in recursive algorithms?;; Key strategies include: ensuring proper base cases in recursive functions, converting recursion to iteration when possible, using memoization to reduce call depth, implementing tail recursion (where supported), using explicit stacks instead of call stack, and breaking large problems into smaller chunks. Additionally, increasing stack size limits (where configurable) can help with legitimate deep recursion needs.