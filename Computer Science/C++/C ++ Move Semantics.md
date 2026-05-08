---
memory: to_finish
tags:
  - learned
language:
  - C++
review-date:
last-reviewed: 2025-09-16
scheda: done
visit-count: 5
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-08-15
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
C++ **Move Semantics**, introduced in C++11, fundamentally solves the ==problem of **inefficient and unnecessary deep copying of resources** within programs, particularly when dealing with temporary objects or when transferring ownership of resources==. <mark style="background: #FF5582A6;">Before</mark> C++11, ==operations like returning large objects by value from functions, or assigning one object to another where both manage dynamic memory e.g., `std::vector`, `std::string`, custom classes with raw pointers, often resulted in expensive **deep copies**==. <mark style="background: #FF5582A6;">A deep copy involves allocating new memory and copying all the data, which is time-consuming and resource-intensive for large objects</mark>.

==Move semantics provides a way to **transfer ownership** of resources from one object to another without performing a copy==. Instead of copying data, it "steals" the resources (e.g., raw pointers to dynamic memory) from the source object, <mark style="background: #BBFABBA6;">leaving the source in a valid but empty/null state</mark>. This is incredibly important for **performance optimization** in C++, allowing for:
1.  **Faster execution:** Avoiding expensive memory allocations and data copies.
2.  **Reduced memory footprint:** No need for temporary duplications of large data structures.
3.  **Efficient resource management:** Especially critical for classes that wrap system resources like file handles, network sockets, or large dynamically allocated buffers, where copying is often impossible or undesirable.

It's a cornerstone of modern C++ for writing highly performant, scalable, and resource-efficient applications, especially in areas like game development, high-frequency trading, and scientific computing, where every millisecond and byte counts.

# **Core Explanation:**
---
**Move semantics** is a C++ feature that allows the<mark style="background: #D2B3FFA6;"> transfer of resources (such as dynamically allocated memory) from one object to another without making a deep copy. Instead of copying the actual data, the source object's resources are "moved" (or "stolen") to the destination object, and the source object is left in a valid, but typically empty or null, state.</mark>

The core mechanisms enabling move semantics are:

1.  **Rvalue References (`&&`):**
    * An ==rvalue reference is a new type of reference that can bind only to **rvalues**== (temporary objects, literals, or expressions whose address cannot be taken, like the result of `x + y`).
    * They allow overloaded functions (like constructors and assignment operators) to differentiate between lvalue arguments (which should be copied) and rvalue arguments (which can be moved from).
2.  **[[C++ Move Constructor]]:**
    * A special constructor (`ClassName(ClassName&& other);`) that<mark style="background: #FF5582A6;"> takes an rvalue reference</mark>.
    * It's invoked <mark style="background: #FF5582A6;">when initializing a new object from an rvalue</mark> (e.g., `ClassName obj = createTemporaryObject();`).
    * It transfers the internal resources (e.g., `data` pointer, `size`) from `other` to `*this`, and then nullifies `other`'s internal pointers to prevent double deletion when `other` is destroyed.
3.  **[[C++ Move Assignment Operator]]:**
    * A special assignment operator (`ClassName& operator=(ClassName&& other);`) that takes an rvalue reference.
    * It's invoked when <mark style="background: #D2B3FFA6;">assigning an rvalue to an existing object</mark> (e.g., `existingObj = createTemporaryObject();`).
    * Similar to the move constructor, it first releases `*this`'s current resources, then transfers `other`'s resources to `*this`, and finally nullifies `other`'s pointers.
4.  **`std::move`:**
    * A standard library utility function (located in `<utility>`) that <mark style="background: #ADCCFFA6;">casts its argument to an rvalue</mark> reference (`static_cast<T&&>(arg)`).
    * **Crucially, `std::move` ==does not *perform* a move itself==.** It merely<mark style="background: #BBFABBA6;"> converts an lvalue into an rvalue reference</mark>, which *enables* a subsequent move constructor or move assignment operator to be called if available. This is used when you want to move from an lvalue object, indicating that you no longer need its resources.

**How it Works (High-Level):**
When the compiler sees an operation involving an rvalue, like returning a temporary object from a function or using `std::move`, it prioritizes calling the move constructor or move assignment operator if they are defined. These special member functions then perform a "shallow copy" of the internal resource pointers/handles and invalidate the source object, effectively moving ownership of the underlying resource without a deep data copy. This makes the operation extremely fast.

If a class does not explicitly define move constructors/assignment operators, the compiler will implicitly generate them *if* certain conditions are met (e.g., no user-defined copy constructor, copy assignment operator, or destructor). However, for classes managing raw resources (like `int*`), the implicitly generated move operations will still perform shallow copies of the pointers, which can lead to problems like double deallocation. Therefore, for resource-managing classes, manually defining move semantics is almost always necessary.

# **Related Concepts:**
---
- [[C++ - Value Categories]]
* **Lvalues and Rvalues:** These are fundamental categories of expressions in C++ that move semantics relies on. **Lvalues** (locator values) are expressions that refer to a persistent object in memory (e.g., variables like `int x; x`). **Rvalues** (right-hand side values) are temporary objects or expressions that don't have a persistent memory location or whose address cannot be taken (e.g., `10`, `x + y`, `createObject()`). Move semantics primarily applies to rvalues as their resources are about to be destroyed anyway, making them ideal candidates for "stealing."
* **Rvalue References (`&&`):** The syntactical enabler for move semantics. They allow functions to specifically target and bind to rvalue expressions, thus distinguishing between "copyable" and "movable" scenarios.
* **Move Constructor:** A specific special member function that allows construction of an object by moving resources from an rvalue. It's one of the two primary mechanisms for move semantics.
* **Move Assignment Operator:** The other primary mechanism for move semantics, allowing assignment to an existing object by moving resources from an rvalue.
* **`std::move`:** A utility that allows you to treat an lvalue as an rvalue, thereby enabling move semantics for that lvalue. It's crucial when you *know* an lvalue's resources can be safely transferred.
* **Perfect Forwarding `std::forward`:** While related to rvalue references, perfect forwarding is about *preserving* the lvalue/rvalue-ness of an argument when forwarding it to another function. It uses universal references (`T&&` in a template context) and `std::forward` to correctly deduce and pass the original argument's value category, which is vital for building generic library components that utilize move semantics.
* **Copy Semantics:** The traditional way of handling object copying, involving deep copies (allocating new memory and copying data). Move semantics is a direct contrast, aiming to avoid these expensive copies.
* **RAII (Resource Acquisition Is Initialization):** A C++ idiom where resource management (acquisition and release) is tied to the lifetime of an object. Move semantics integrates perfectly with RAII, as move operations correctly transfer resource ownership, ensuring that resources are always properly deallocated by the final owner's destructor. Smart pointers (like `std::unique_ptr`) are prime examples of RAII and internally use move semantics.
* **Return Value Optimization (RVO) / Named Return Value Optimization (NRVO):** Compiler optimizations that can eliminate the need for copy or move constructors when returning objects by value from functions. While related to avoiding copies, RVO/NRVO are compiler-specific and not a language feature that *guarantees* move semantics. Move semantics is a fallback when RVO/NRVO cannot be applied.

# **Examples:**
---

```cpp
#include <iostream>
#include <vector>   // std::vector uses move semantics internally
#include <string>   // std::string uses move semantics internally
#include <utility>  // For std::move

// A custom class that manages a dynamically allocated integer array
// We'll implement copy and move semantics to demonstrate their difference.
class MyBuffer {
public:
    int* data;
    size_t size;

    // Constructor
    MyBuffer(size_t s) : size(s) {
        data = new int[size];
        std::cout << "MyBuffer Constructor: Allocated " << size << " integers at " << data << std::endl;
        for (size_t i = 0; i < size; ++i) {
            data[i] = static_cast<int>(i);
        }
    }

    // Destructor
    ~MyBuffer() {
        if (data) {
            delete[] data;
            data = nullptr;
            std::cout << "MyBuffer Destructor: Deallocated memory at " << data << std::endl;
        } else {
            std::cout << "MyBuffer Destructor: Already empty." << std::endl;
        }
    }

    // Copy Constructor (Deep Copy)
    MyBuffer(const MyBuffer& other) : size(other.size) {
        data = new int[size]; // Allocate NEW memory
        std::copy(other.data, other.data + other.size, data); // Copy all data
        std::cout << "MyBuffer Copy Constructor: Deep copied " << size << " integers from " << other.data << " to " << data << std::endl;
    }

    // Copy Assignment Operator (Deep Copy)
    MyBuffer& operator=(const MyBuffer& other) {
        if (this != &other) {
            delete[] data; // Release current resources
            size = other.size;
            data = new int[size]; // Allocate NEW memory
            std::copy(other.data, other.data + other.size, data); // Copy all data
            std::cout << "MyBuffer Copy Assignment: Deep copied " << size << " integers." << std::endl;
        }
        return *this;
    }

    // !!! Move Constructor !!!
    // Takes an rvalue reference (&&). 'noexcept' is good practice.
    MyBuffer(MyBuffer&& other) noexcept 
        : data(other.data), // 'Steal' the data pointer
          size(other.size)   // 'Steal' the size
    {
        // Set the source object to an empty state
        // This is crucial: when 'other' is destructed, it won't delete the memory
        // that 'this' (the new object) now owns.
        other.data = nullptr; 
        other.size = 0;
        std::cout << "MyBuffer Move Constructor: Moved resources from " << other.data << " to " << data << std::endl;
    }

    // !!! Move Assignment Operator !!!
    // Takes an rvalue reference (&&). 'noexcept' is good practice.
    MyBuffer& operator=(MyBuffer&& other) noexcept {
        if (this != &other) { // Self-assignment check
            delete[] data; // Release current resources of 'this'

            // 'Steal' resources from 'other'
            data = other.data;
            size = other.size;

            // Nullify 'other' to prevent it from deleting the moved resources
            other.data = nullptr;
            other.size = 0;
            std::cout << "MyBuffer Move Assignment: Moved resources from " << other.data << " to " << data << std::endl;
        }
        return *this;
    }

    void print() const {
        std::cout << "MyBuffer (Size: " << size << ", Data Ptr: " << data << "): [";
        if (data) {
            for (size_t i = 0; i < size; ++i) {
                std::cout << data[i] << (i == size - 1 ? "" : ", ");
            }
        } else {
            std::cout << "EMPTY";
        }
        std::cout << "]" << std::endl;
    }
};

// Function returning a MyBuffer by value
MyBuffer createBuffer(size_t s) {
    std::cout << "\n--- Inside createBuffer() ---" << std::endl;
    MyBuffer temp(s); // Constructor for 'temp'
    std::cout << "Temporary object 'temp' in createBuffer: "; temp.print();
    std::cout << "--- Exiting createBuffer() ---" << std::endl;
    return temp; // This is an rvalue. Will trigger move constructor for return value optimization.
                 // If RVO/NRVO is not applied, a move constructor will be used.
}

int main() {
    std::cout << "=== Demonstrating Move Semantics ===" << std::endl;

    // Scenario 1: Initializing a new object from a temporary (rvalue)
    // The result of createBuffer(5) is an rvalue.
    // This will likely use the Move Constructor to initialize `buf1`.
    std::cout << "\n--- Creating buf1 (from temporary) ---" << std::endl;
    MyBuffer buf1 = createBuffer(5); 
    std::cout << "buf1 after creation: "; buf1.print();

    // Scenario 2: Assigning a temporary (rvalue) to an existing object
    // The result of createBuffer(3) is an rvalue.
    // This will use the Move Assignment Operator to assign to `buf2`.
    std::cout << "\n--- Creating buf2 and assigning from temporary ---" << std::endl;
    MyBuffer buf2(2); // Constructor for buf2
    std::cout << "buf2 before assignment: "; buf2.print();
    buf2 = createBuffer(3); 
    std::cout << "buf2 after assignment: "; buf2.print();

    // Scenario 3: Explicitly moving from an lvalue using std::move
    // `buf3` is an lvalue. Normally, `buf4 = buf3;` would call copy assignment.
    // But `std::move(buf3)` converts `buf3` into an rvalue reference,
    // triggering the move assignment operator.
    std::cout << "\n--- Explicitly moving from an lvalue (std::move) ---" << std::endl;
    MyBuffer buf3(4); // Constructor for buf3
    std::cout << "buf3 before move: "; buf3.print();
    
    MyBuffer buf4(1); // Constructor for buf4
    std::cout << "buf4 before move assignment: "; buf4.print();

    std::cout << "Performing buf4 = std::move(buf3);" << std::endl;
    buf4 = std::move(buf3); // Calls MyBuffer::operator=(MyBuffer&&)
    
    std::cout << "buf3 after explicit move: "; buf3.print(); // buf3 is now empty/invalidated
    std::cout << "buf4 after explicit move: "; buf4.print(); // buf4 now owns original buf3's data

    // Scenario 4: Copying for comparison
    std::cout << "\n--- Copying from an lvalue (for comparison) ---" << std::endl;
    MyBuffer buf5(2); // Constructor
    std::cout << "buf5 before copy: "; buf5.print();
    MyBuffer buf6 = buf5; // Calls MyBuffer::MyBuffer(const MyBuffer&) - Copy Constructor
    std::cout << "buf5 after copy: "; buf5.print(); // buf5 is unchanged
    std::cout << "buf6 after copy: "; buf6.print(); // buf6 has its own separate deep copy

    std::cout << "\n=== End of Main ===" << std::endl;
    // Destructors will be called for buf1, buf2, buf3 (empty), buf4, buf5, buf6.
    return 0;
}
````

# **Flashcards:**
---

What is C++ Move Semantics primarily designed to optimize?;;It optimizes resource management by allowing efficient transfer of resources (instead of deep copying) from temporary objects or objects explicitly marked for moving.

What are the two main special member functions that enable move semantics?;;The Move Constructor (ClassName(ClassName&&)) and the Move Assignment Operator (ClassName& operator=(ClassName&&)).

What does std::move do?;;std::move casts an lvalue into an rvalue reference, enabling a subsequent move operation (constructor or assignment) to be selected, but it does not perform the move itself.