---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-10-07
scheda: done
visit-count: 3
confidence-level: 2.5
consecutive-correct: 3
last-struggle-date: ""
cssclasses:
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

Recursion solves problems that can be broken down into smaller, similar subproblems. It's particularly powerful for tasks involving <mark style="background: #BBFABBA6;">tree-like structures, mathematical sequences, divide-and-conquer algorithms, and problems with self-similar patterns</mark>. Recursion is essential in computer science because it provides an elegant and intuitive way to solve complex problems by reducing them to simpler versions of themselves, making code more readable and maintainable for certain problem domains.

# **Core Explanation:**
---

Recursion is a programming technique where a function calls itself to solve a problem. Every recursive function must have two essential components:

>1. **Base case**: A condition that stops the recursion and returns a result without further recursive calls
>2. **Recursive case**: The function calls itself with modified parameters, moving toward the base case

When a <mark style="background: #FF5582A6;">recursive function is called, each call is added to the call stack</mark>. The function continues calling itself until it reaches the base case, then returns values back up the stack, unwinding the recursion. This creates a natural "divide and conquer" approach where complex problems are broken into smaller, manageable pieces.

Key characteristics:
- Self-referential function calls
- Stack-based execution model
- Natural fit for tree traversal and mathematical sequences
- Can be more intuitive than iterative solutions for certain problems
- May have higher memory overhead due to call stack usage

# **Related Concepts:**
---

**Iteration**: The alternative to recursion using loops. While recursion uses function calls, iteration uses explicit loop constructs. Some problems can be solved with either approach.

**[[Call Stack]]**: The memory structure that tracks function calls. Recursion heavily relies on the call stack, which can lead to stack overflow if recursion depth is too large.

**Dynamic Programming**: Often uses recursion with memoization to solve optimization problems by storing results of subproblems.

**Tree Traversal**: Recursion is the natural approach for navigating tree structures (binary trees, file systems, etc.).

**Divide and Conquer**: A problem-solving paradigm that recursively breaks problems into smaller subproblems, solves them independently, and combines results.

**Mathematical Induction**: The logical foundation similar to recursion - prove base case, then prove if true for n, it's true for n+1.

# **Examples:**
---
## C++ Examples:

```cpp
#include <iostream>
#include <vector>
using namespace std;

// Example 1: Factorial calculation
// Demonstrates basic recursion with mathematical sequence
int factorial(int n) {
    // Base case: factorial of 0 and 1 is 1
    // This stops the recursion from continuing infinitely
    if (n <= 1) {
        return 1;
    }
    
    // Recursive case: n! = n * (n-1)!
    // Function calls itself with a smaller problem (n-1)
    // Each call waits for the result of the smaller problem
    return n * factorial(n - 1);
}

// Example 2: Fibonacci sequence
// Shows how recursion naturally maps to mathematical definitions
int fibonacci(int n) {
    // Base cases: first two Fibonacci numbers are defined
    // Without these, the function would call itself indefinitely
    if (n <= 1) {
        return n;
    }
    
    // Recursive case: F(n) = F(n-1) + F(n-2)
    // This creates a binary tree of recursive calls
    // Note: This is inefficient due to repeated calculations
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// Example 3: Binary tree traversal
struct TreeNode {
    int val;
    TreeNode* left;
    TreeNode* right;
    TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
};

// Inorder traversal: left -> root -> right
void inorderTraversal(TreeNode* root) {
    // Base case: empty node, nothing to process
    // This handles leaf nodes and prevents null pointer access
    if (root == nullptr) {
        return;
    }
    
    // Recursive case: process left subtree first
    inorderTraversal(root->left);
    
    // Process current node
    cout << root->val << " ";
    
    // Recursive case: process right subtree
    inorderTraversal(root->right);
}

// Example 4: Array sum using divide and conquer
int arraySum(vector<int>& arr, int start, int end) {
    // Base case: single element or empty range
    // Single element is the sum of itself
    if (start == end) {
        return arr[start];
    }
    
    // Base case: empty range
    if (start > end) {
        return 0;
    }
    
    // Recursive case: divide array in half and sum both parts
    // This demonstrates divide and conquer approach
    int mid = start + (end - start) / 2;
    int leftSum = arraySum(arr, start, mid);
    int rightSum = arraySum(arr, mid + 1, end);
    
    // Combine results from both halves
    return leftSum + rightSum;
}
````

## JavaScript Examples:

```javascript
// Example 1: Factorial calculation
// Same logic as C++ but with JavaScript syntax
function factorial(n) {
    // Base case: stops recursion when n reaches 0 or 1
    // Essential to prevent infinite recursion
    if (n <= 1) {
        return 1;
    }
    
    // Recursive case: break down problem into smaller subproblem
    // Each recursive call reduces the problem size by 1
    return n * factorial(n - 1);
}

// Example 2: String reversal using recursion
function reverseString(str) {
    // Base case: empty string or single character
    // Cannot be reversed further, so return as is
    if (str.length <= 1) {
        return str;
    }
    
    // Recursive case: take last character and add reversed substring
    // This builds the reversed string from the end backwards
    return str[str.length - 1] + reverseString(str.slice(0, -1));
}

// Example 3: Object deep cloning
function deepClone(obj) {
    // Base case: primitive values (null, numbers, strings, booleans)
    // These can be returned directly without further processing
    if (obj === null || typeof obj !== 'object') {
        return obj;
    }
    
    // Handle arrays specifically
    if (Array.isArray(obj)) {
        // Recursive case: clone each element in the array
        // map() applies deepClone to each element
        return obj.map(item => deepClone(item));
    }
    
    // Handle objects
    const cloned = {};
    // Recursive case: clone each property value
    // Object.keys() gets all property names
    for (let key of Object.keys(obj)) {
        cloned[key] = deepClone(obj[key]);
    }
    
    return cloned;
}

// Example 4: Nested array flattening
function flattenArray(arr) {
    let result = [];
    
    for (let item of arr) {
        // Base case: item is not an array, add it directly
        if (!Array.isArray(item)) {
            result.push(item);
        } else {
            // Recursive case: item is an array, flatten it first
            // Use spread operator to add all elements from flattened array
            result.push(...flattenArray(item));
        }
    }
    
    return result;
}

// Example 5: Binary search (recursive implementation)
function binarySearch(arr, target, left = 0, right = arr.length - 1) {
    // Base case: target not found (search space exhausted)
    if (left > right) {
        return -1;
    }
    
    // Find middle point to divide search space
    let mid = Math.floor((left + right) / 2);
    
    // Base case: target found at middle position
    if (arr[mid] === target) {
        return mid;
    }
    
    // Recursive case: target is in left half
    if (arr[mid] > target) {
        return binarySearch(arr, target, left, mid - 1);
    }
    
    // Recursive case: target is in right half
    return binarySearch(arr, target, mid + 1, right);
}

// Usage examples:
console.log(factorial(5)); // Output: 120
console.log(reverseString("hello")); // Output: "olleh"
console.log(flattenArray([1, [2, 3], [4, [5, 6]]])); // Output: [1, 2, 3, 4, 5, 6]
```

# **Flashcards:**
---

What are the two essential components every recursive function must have?;; Base case (stops recursion and returns result) and Recursive case (function calls itself with modified parameters moving toward base case)

What is the primary risk of recursive functions and how can it be avoided?;; Stack overflow from infinite recursion. Avoided by ensuring base cases are properly defined and reachable, and that recursive calls move toward the base case.

How does recursion differ from iteration in terms of memory usage?;; Recursion uses more memory due to call stack overhead (each recursive call adds a frame to the stack), while iteration typically uses constant memory with loop variables.

What is tail recursion and why is it important?;; Tail recursion occurs when the recursive call is the last operation in the function. It's important because some compilers can optimize it to avoid stack overflow by reusing the current stack frame.

Give an example of a problem that is naturally suited for recursion.;; Tree traversal, factorial calculation, Fibonacci sequence, divide-and-conquer algorithms, or any problem with self-similar subproblems.

What happens in the call stack during a recursive function execution?;; Each recursive call creates a new stack frame containing local variables and parameters. Frames accumulate until base case is reached, then results are returned and frames are popped off the stack in reverse order.