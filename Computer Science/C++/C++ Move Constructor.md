---
memory: to_finish
tags:
  - learned
language:
  - C++
review-date:
last-reviewed: 2025-10-10
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-09-02
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
The C++ **move constructor** ==addresses the inefficiency of **deep copying** when an object is being initialized from a temporary (rvalue) object==. <mark style="background: #ADCCFFA6;">Before C++11</mark> and the introduction of move semantics, <mark style="background: #ADCCFFA6;">initializing a new object from a temporary one that managed dynamic resources</mark>, like a `std::vector` or a custom class with a dynamically allocated array, <mark style="background: #ADCCFFA6;">would trigger the copy constructor.</mark> 
This meant:
1.  Allocating new memory for the new object.
2.  Copying all the data from the temporary object's resources to the new object's resources.

This process is ==highly inefficient for large objects,== as it involves significant memory allocation and data transfer. The move constructor provides a mechanism to **transfer ownership of resources** from the temporary source object to the newly constructed object, rather than copying them. This "resource stealing" avoids redundant allocations and copies, leading to **significant performance improvements** for operations involving temporary objects, function return values, and efficient container manipulations. It's a cornerstone of modern C++ for writing high-performance and resource-efficient code.

# **Core Explanation:**
---
The **move constructor** is a special member function in C++ (introduced in C++11) that constructs a ==new object by "moving" resources from an existing **rvalue** object==. Instead of performing a deep copy, it efficiently transfers ownership of dynamic resources from the source to the newly created object.

Its signature typically looks like `ClassName(ClassName&& other);`, where `ClassName&& other` is an **rvalue reference** to the source object.

**Key Characteristics and How it Works:**

1.  **Rvalue Reference Parameter (`&&`):** Like the move assignment operator, the move constructor takes an rvalue reference as its parameter. ==This means it will be invoked when initializing an object from a temporary object (an rvalue) or an object explicitly cast to an rvalue using `std::move()`.==
2.  **Resource Transfer (Stealing):** When invoked, the move constructor:
    * ==Initializes the new object's resource pointers/members by copying them directly from the `other` (source) object. This is a shallow copy of the resource *pointers*, not a deep copy of the *data*.==
    * ==Sets `other`'s resource pointers/members to a null or default state==. <mark style="background: #BBFABBA6;">This is crucial: it ensures that when `other`'s destructor is called</mark> (as it's often a temporary object about to be destroyed), it doesn't attempt to deallocate the resources that have now been taken by the newly constructed object.
3.  **No Deep Copy:** The key advantage is that it avoids the overhead of allocating new memory and copying the actual data. It's simply transferring pointers and potentially sizes.
4.  **`noexcept` Specifier:** Move constructors are typically declared `noexcept`. This is important because throwing an exception during a move operation can leave objects in an indeterminate state and can prevent certain standard library containers from using move semantics, forcing them to fall back to less efficient copy operations.
5.  **Implicit Declaration:** If a user-defined move constructor is not provided, the compiler will implicitly declare one if there are no user-declared copy constructor, copy assignment operator, or destructor. However, this implicitly declared move constructor performs member-wise move construction. If a class manages raw pointers, the implicitly generated move constructor will only shallow-copy the pointer, leading to double deallocations or memory leaks if not handled correctly. Therefore, for classes managing raw resources, it's almost always necessary to explicitly define the move constructor.

# **Related Concepts:**
---

* **Move Assignment Operator:** The move constructor is part of the broader **move semantics** feature, and its counterpart for assignment is the move assignment operator. Both enable resource transfer, but the constructor is for *initialization* of a new object, while the assignment operator is for *assigning* to an existing object.
* **Copy Constructor:** This is the direct counterpart to the move constructor. The copy constructor (`ClassName(const ClassName& other);`) performs a **deep copy**, creating new resources and copying data. The compiler chooses between the copy and move constructors based on whether the source object used for initialization is an lvalue or an rvalue.
* **Rvalue References (`&&`):** These are fundamental to both move constructors and move assignment operators. An rvalue reference parameter allows a function or constructor to bind specifically to temporary objects or objects designated for move operations, enabling the "resource stealing" optimization.
* **`std::move`:** This utility function (`static_cast<T&&>`) is used to explicitly cast an lvalue into an rvalue reference. This forces the compiler to consider move semantics for that object, even if it's not a temporary. It's crucial when you want to move from an lvalue.
* **RAII (Resource Acquisition Is Initialization):** The move constructor (and move assignment operator) are often designed in conjunction with RAII. By defining these members correctly, you ensure that resource ownership is safely transferred, and resources are always properly managed (acquired in constructors, released in destructors) even across move operations.
* **Rule of Five (or Three/Zero):** The move constructor is one of the "five" special member functions (along with destructor, copy constructor, copy assignment operator, and move assignment operator). The rule suggests that if you define any of these for a class that manages resources, you should typically define all of them (or rely on the compiler's implicit generation only if it truly does what you want, which is rare for resource-managing classes) to ensure correct behavior and avoid issues like double deallocation or memory leaks.

# **Examples:**
---

```cpp
#include <iostream>
#include <vector>   // For std::vector which has move semantics built-in
#include <utility>  // For std::move

// A simple class that manages a dynamically allocated integer array
class MyVector {
public:
    int* data;
    size_t size;

    // Constructor
    MyVector(size_t s) : size(s) {
        data = new int[size]; // Allocate dynamic memory
        std::cout << "Constructor: Allocated " << size << " integers." << std::endl;
        for (size_t i = 0; i < size; ++i) {
            data[i] = static_cast<int>(i); // Initialize data
        }
    }

    // Destructor
    ~MyVector() {
        if (data) { // Check to prevent double deletion or deleting null
            delete[] data; // Deallocate dynamic memory
            data = nullptr; // Good practice to nullify pointer after deletion
            std::cout << "Destructor: Deallocated memory." << std::endl;
        }
    }

    // Copy Constructor (Deep Copy)
    MyVector(const MyVector& other) : size(other.size) {
        data = new int[size]; // Allocate new memory for the copy
        std::copy(other.data, other.data + other.size, data); // Copy data
        std::cout << "Copy Constructor: Deep copied " << size << " integers." << std::endl;
    }

    // Copy Assignment Operator (Deep Copy - omitted for brevity, but typically needed)
    MyVector& operator=(const MyVector& other) {
        if (this != &other) {
            delete[] data;
            size = other.size;
            data = new int[size];
            std::copy(other.data, other.data + other.size, data);
            std::cout << "Copy Assignment: Deep copied " << size << " integers." << std::endl;
        }
        return *this;
    }

    // !!! MOVE CONSTRUCTOR !!!
    // Takes an rvalue reference (&&) indicating it can "steal" resources
    MyVector(MyVector&& other) noexcept : data(nullptr), size(0) { // Initialize to null/zero
        // 1. "Steal" the resources from 'other'
        //    Directly copy the pointer and size from the source.
        data = other.data; 
        size = other.size;

        // 2. Nullify 'other's pointer and reset its size
        //    This is crucial. 'other' is typically a temporary object.
        //    When 'other's destructor runs, it won't try to delete the memory
        //    that has now been taken by 'this' (the newly constructed object).
        other.data = nullptr; 
        other.size = 0; 

        std::cout << "Move Constructor: Resources moved. (Original source now empty)" << std::endl;
    }

    // Move Assignment Operator (omitted for brevity, but typically needed)
    MyVector& operator=(MyVector&& other) noexcept {
        if (this != &other) {
            delete[] data;
            data = other.data;
            size = other.size;
            other.data = nullptr;
            other.size = 0;
            std::cout << "Move Assignment: Resources moved." << std::endl;
        }
        return *this;
    }

    void printData() const {
        if (data && size > 0) {
            std::cout << "Data: [";
            for (size_t i = 0; i < size; ++i) {
                std::cout << data[i] << (i == size - 1 ? "" : ", ");
            }
            std::cout << "]";
        } else {
            std::cout << "Data: [EMPTY]";
        }
        std::cout << std::endl;
    }
};

// Function that returns a MyVector object by value.
// This will often create a temporary object (an rvalue) that can be moved.
MyVector createAndReturnVector(size_t s) {
    MyVector temp(s); // Constructor
    std::cout << "Inside createAndReturnVector, temp: "; temp.printData();
    return temp; // Return by value: triggers move constructor (or RVO/NRVO)
}

int main() {
    std::cout << "--- Scenario 1: Initialization with a temporary (Move Constructor) ---" << std::endl;
    // Calling a function that returns MyVector by value results in a temporary (rvalue).
    // This temporary will be used to move-construct 'vec1'.
    MyVector vec1 = createAndReturnVector(4); // Calls move constructor (or RVO/NRVO)
    std::cout << "vec1 after initialization: "; vec1.printData();
    // The temporary object returned by createAndReturnVector() was moved from.
    // Its destructor will be called, but it's in an empty state, so no double deletion.

    std::cout << "\n--- Scenario 2: Explicit std::move from an lvalue ---" << std::endl;
    MyVector original_vec(2); // Constructor
    std::cout << "Original vec: "; original_vec.printData();

    // Using std::move to explicitly cast original_vec (an lvalue) to an rvalue reference.
    // This forces the move constructor to be used for 'moved_vec'.
    MyVector moved_vec = std::move(original_vec); // Calls move constructor
    std::cout << "Original vec after move: "; original_vec.printData(); // original_vec is now empty!
    std::cout << "Moved vec: "; moved_vec.printData(); // moved_vec now owns original_vec's resources

    std::cout << "\n--- Scenario 3: Copy Constructor (for comparison) ---" << std::endl;
    MyVector source_vec(3); // Constructor
    std::cout << "Source vec: "; source_vec.printData();

    // Initializing 'copied_vec' from 'source_vec' (an lvalue) calls the copy constructor.
    MyVector copied_vec = source_vec; // Calls copy constructor
    std::cout << "Source vec after copy: "; source_vec.printData(); // source_vec is unchanged
    std::cout << "Copied vec: "; copied_vec.printData(); // copied_vec has its own deep copy

    std::cout << "\nEnd of main scope." << std::endl;
    // Destructors will be called for vec1, moved_vec, original_vec (now empty), copied_vec, source_vec.
    return 0;
}
```

# **Flashcards:**
---

What is the primary purpose of the C++ move constructor?;;To efficiently construct a new object by transferring resources from a temporary (rvalue) source object, avoiding deep copies.

What is the typical signature of a move constructor?;;ClassName(ClassName&& other); (It takes an rvalue reference).

How does the move constructor ensure proper resource management after transferring from the source?;;It sets the source object's resource pointers (e.g., data) to nullptr to prevent its destructor from deallocating resources now owned by the new object.