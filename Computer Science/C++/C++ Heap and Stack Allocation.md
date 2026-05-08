---
memory: to_finish
tags:
  - learned
language:
  - C++
review-date: ""
last-reviewed: 2025-07-11
keywords:
  - heap
  - stack
  - memory allocation
  - new keyword
  - delete
  - malloc
  - free
  - scope
  - lifetime
scheda: done
visit-count: 4
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-06-29
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
--- 
In C++, memory allocation primarily occurs in two distinct regions: the **stack** and the **heap (or free store)**. Understanding their differences is crucial for efficient memory management and avoiding common pitfalls like memory leaks or stack overflows.

**Stack Allocation:**
The stack is a region of memory used for static memory allocation. Variables declared with [[C++ - Automatic Storage Duration]] (i.e., local variables within functions) are typically allocated on the stack. This allocation is managed automatically by the compiler. ==When a function is called, a new "stack frame" is created, containing space for its local variables and function arguments.== When the function returns, its stack frame is popped off, and the memory is automatically deallocated.

* **Characteristics:**
    * **Automatic Management:** Memory is automatically allocated and deallocated.
    * **Fast:** Allocation and deallocation are very fast due to the LIFO (Last-In, First-Out) nature of the stack.
    * **Limited Size:** The stack has a relatively small, fixed size (typically a few megabytes), which can lead to stack overflow if too much memory is requested (e.g., deeply recursive functions or very large local arrays).
    * **Scope-Bound Lifetime:** The lifetime of variables on the stack is tied to their scope; they are destroyed when their scope ends.

**Heap Allocation (Dynamic Memory Allocation):**
The heap is a region of memory used for dynamic memory allocation. Unlike the stack, memory on the heap must be explicitly allocated and deallocated by the programmer. This allows for flexible memory management, where the size and lifetime of objects are not restricted by function scope. In C++, `new` and `delete` operators are used for heap allocation and deallocation, respectively. `malloc` and `free` (from C's `stdlib.h`) can also be used.

* **Characteristics:**
    * **Manual Management:** Memory must be explicitly allocated with `new` (or `malloc`) and deallocated with `delete` (or `free`). Failure to deallocate leads to memory leaks.
    * **Slower:** Allocation and deallocation on the heap are generally slower than on the stack due to the overhead of searching for available memory blocks.
    * **Flexible Size:** The heap is much larger than the stack and can accommodate large objects or data structures whose size is unknown at compile time.
    * **Program-Controlled Lifetime:** The lifetime of objects on the heap is independent of their scope; they persist until explicitly deleted.

# **Related Concepts:**
---
* **Memory Leaks:** Occur when dynamically allocated memory is no longer referenced but has not been deallocated, leading to a gradual depletion of available memory.
* **Dangling Pointers:** Pointers that point to a memory location that has been deallocated, leading to undefined behavior if dereferenced.
* **Double Free:** Attempting to deallocate the same block of memory twice, also leading to undefined behavior.
* **Stack Overflow:** Occurs when the program attempts to use more stack space than is available, typically due to excessively deep recursion or very large local variables.
* **RAII (Resource Acquisition Is Initialization):** A C++ programming idiom that ties the lifetime of a resource (like dynamically allocated memory) to the lifetime of an object. Smart pointers (like `std::unique_ptr` and `std::shared_ptr`) are excellent examples of RAII for managing heap memory automatically.
* **Static Memory Allocation:** Memory allocated at compile time with a fixed size and lifetime for global or static variables. Not to be confused with stack allocation, though both are forms of automatic memory management (to some extent).
# **Examples:**
--- 
```cpp
#include <iostream>
#include <vector>
#include <memory> // For std::unique_ptr and std::shared_ptr

// Function demonstrating stack allocation
void stackAllocationExample() {
    // 'x' is allocated on the stack. Its lifetime is tied to this function's scope.
    int x = 10;
    std::cout << "Stack variable x: " << x << " (Address: " << &x << ")" << std::endl;

    // 'arr' is a fixed-size array allocated on the stack.
    int arr[5] = {1, 2, 3, 4, 5};
    std::cout << "Stack array arr[0]: " << arr[0] << " (Address: " << &arr[0] << ")" << std::endl;

    // When this function returns, 'x' and 'arr' are automatically deallocated.
}

// Function demonstrating heap allocation using new/delete
int* createIntOnHeap() {
    // 'ptr' is a pointer allocated on the stack.
    // The integer it points to (*ptr) is allocated on the heap.
    int* ptr = new int; // Allocate memory for an integer on the heap
    *ptr = 25; // Store a value in the heap-allocated memory
    std::cout << "Heap variable *ptr: " << *ptr << " (Address: " << ptr << ")" << std::endl;
    return ptr; // Return the address of the heap-allocated memory
} // 'ptr' (the local pointer) is deallocated here, but the memory it pointed to on the heap remains.

// Function demonstrating heap allocation with an array using new[]/delete[]
double* createDoubleArrayOnHeap(int size) {
    // 'arr_ptr' is a pointer allocated on the stack.
    // The array it points to is allocated on the heap.
    double* arr_ptr = new double[size]; // Allocate an array of 'size' doubles on the heap
    for (int i = 0; i < size; ++i) {
        arr_ptr[i] = i * 1.5;
    }
    std::cout << "Heap array arr_ptr[0]: " << arr_ptr[0] << " (Address: " << arr_ptr << ")" << std::endl;
    return arr_ptr;
}

// Function demonstrating heap allocation with std::vector (uses heap internally)
void vectorHeapAllocationExample() {
    // std::vector allocates its elements on the heap, but the vector object itself is on the stack.
    std::vector<int> myVector; // 'myVector' object is on the stack
    myVector.push_back(100);
    myVector.push_back(200);
    std::cout << "Vector element myVector[0]: " << myVector[0] << std::endl;
    // When 'myVector' goes out of scope, its destructor automatically deallocates the heap memory.
}

// Function demonstrating smart pointers (std::unique_ptr) for RAII
void uniquePtrExample() {
    // 'uPtr' is a smart pointer allocated on the stack.
    // The integer it manages is allocated on the heap.
    std::unique_ptr<int> uPtr(new int(50)); // Allocate and manage an int on the heap
    std::cout << "Unique pointer value: " << *uPtr << " (Address: " << uPtr.get() << ")" << std::endl;
    // When 'uPtr' goes out of scope, the memory it manages is automatically deallocated.
}

// Function demonstrating smart pointers (std::shared_ptr)
void sharedPtrExample() {
    std::shared_ptr<double> sPtr1 = std::make_shared<double>(75.5); // Allocate and manage a double on the heap
    std::cout << "Shared pointer 1 value: " << *sPtr1 << " (Address: " << sPtr1.get() << ")" << std::endl;
    std::cout << "Shared pointer 1 use count: " << sPtr1.use_count() << std::endl;

    std::shared_ptr<double> sPtr2 = sPtr1; // Copying the shared_ptr, increments reference count
    std::cout << "Shared pointer 2 value: " << *sPtr2 << " (Address: " << sPtr2.get() << ")" << std::endl;
    std::cout << "Shared pointer 1 use count after copy: " << sPtr1.use_count() << std::endl;
    // When the last shared_ptr managing the resource goes out of scope, the memory is automatically deallocated.
}


int main() {
    std::cout << "--- Stack Allocation Example ---" << std::endl;
    stackAllocationExample(); // x and arr are deallocated when this function returns.
    std::cout << "--- End Stack Allocation Example ---" << std::endl << std::endl;

    std::cout << "--- Heap Allocation Example (Manual) ---" << std::endl;
    int* heapVarPtr = createIntOnHeap();
    // At this point, heapVarPtr points to memory on the heap.
    // If we don't 'delete' it, it's a memory leak!
    std::cout << "Back in main: *heapVarPtr: " << *heapVarPtr << " (Address: " << heapVarPtr << ")" << std::endl;
    delete heapVarPtr; // Explicitly deallocate the memory
    heapVarPtr = nullptr; // Good practice to set to nullptr after deleting

    double* heapArrayPtr = createDoubleArrayOnHeap(3);
    // Again, need to delete the array
    std::cout << "Back in main: heapArrayPtr[0]: " << heapArrayPtr[0] << " (Address: " << heapArrayPtr << ")" << std::endl;
    delete[] heapArrayPtr; // Use delete[] for arrays
    heapArrayPtr = nullptr;

    std::cout << "--- End Heap Allocation Example (Manual) ---" << std::endl << std::endl;

    std::cout << "--- std::vector Heap Allocation Example ---" << std::endl;
    vectorHeapAllocationExample(); // Vector's heap memory is deallocated automatically
    std::cout << "--- End std::vector Heap Allocation Example ---" << std::endl << std::endl;

    std::cout << "--- std::unique_ptr Example (RAII) ---" << std::endl;
    uniquePtrExample(); // unique_ptr automatically deallocates heap memory
    std::cout << "--- End std::unique_ptr Example (RAII) ---" << std::endl << std::endl;

    std::cout << "--- std::shared_ptr Example (RAII) ---" << std::endl;
    sharedPtrExample(); // shared_ptr automatically deallocates heap memory
    std::cout << "--- End std::shared_ptr Example (RAII) ---" << std::endl << std::endl;

    // Example of potential stack overflow (DO NOT UNCOMMENT IN PRODUCTION CODE WITHOUT CAUTION)
    // void causeStackOverflow(int depth) {
    //     int localArray[1000]; // Large array on stack
    //     std::cout << "Depth: " << depth << std::endl;
    //     causeStackOverflow(depth + 1);
    // }
    // try {
    //     causeStackOverflow(0);
    // } catch (const std::bad_alloc& e) { // This catch might not always work for stack overflows
    //     std::cerr << "Caught exception: " << e.what() << std::endl;
    // } catch (...) {
    //     std::cerr << "Caught an unknown exception (likely stack overflow)" << std::endl;
    // }

    return 0;
}
```

# Flashcards:
---
What is the primary difference in memory management between stack and heap allocation?;; Stack memory is automatically managed (allocated and deallocated by the compiler), while heap memory requires manual management (explicit allocation with new/malloc and deallocation with delete/free).
<!--SR:!2025-06-18,4,270-->

What are the advantages and disadvantages of stack allocation?;; Advantages: Fast allocation/deallocation, automatic management. Disadvantages: Limited size, lifetime tied to scope, risk of stack overflow.

 When should you use heap allocation in C++?;; Use heap allocation when the size of the data is not known at compile time, when the data needs to persist beyond the scope of a function, or when dealing with very large objects that would exceed stack limits. Prefer smart pointers (std::unique_ptr, std::shared_ptr) to manage heap memory to prevent leaks.
<!--SR:!2025-06-17,3,250-->