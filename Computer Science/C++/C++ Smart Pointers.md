---
memory: to_finish
tags:
 - learned
language:
 - C++
review-date:
last-reviewed: 2025-08-21
scheda: done
visit-count: 5
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-07-22

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

Smart pointers in C++ fundamentally ==solve the problem of manual memory management==, ==specifically dealing with dynamically allocated objects. In C++, when you allocate memory using `new`, it's your responsibility to deallocate it using `delete` to prevent memory leaks. Forgetting to `delete` or doing it incorrectly (e.g., double-deleting) leads to serious bugs and instability.==

==Smart pointers automate this process by providing== [[C++ RAII (Resource Acquisition Is Initialization) ]]==semantics. This means that the memory is automatically deallocated when the smart pointer object goes out of scope==. This is crucial in C++ because it allows developers to write safer, more robust code by significantly reducing the risk of memory leaks and dangling pointers, which are common sources of errors in programs that rely heavily on dynamic memory. They are important in computer science as they represent a common pattern for managing resources with deterministic lifetimes, not just memory.

# **Core Explanation:**

---
A smart pointer ==is an object that acts like a normal pointer but also manages the memory it points to. It's essentially a [[Wrapper]] around a raw pointer that handles the allocation and deallocation of the pointed-to memory automatically.== <mark style="background:D2B3FFA6;">This automatic management is achieved through the RAII principle: the resource (memory) is acquired during object construction and released during object destruction.</mark>

Key characteristics of smart pointers:

- **Automatic Memory Management:** They automatically deallocate the memory they own when they are destroyed (e.g., when they go out of scope).
- **Ownership Semantics:** Each type of smart pointer clearly defines its ownership model (exclusive, shared, non-owning).
- **Behave Like Raw Pointers:** They overload operators like `*` and `->` to allow access to the underlying object as if they were raw pointers.
- **Prevent Memory Leaks:** By automating deallocation, they greatly reduce the chances of memory leaks.
- **Prevent Dangling Pointers:** Some smart pointers (like `unique_ptr`) prevent multiple pointers from owning the same memory, thus reducing the risk of dangling pointers after one owner deallocates the memory.

There are three main types of smart pointers in the C++ Standard Library:

>1. **`std::unique_ptr`**: Provides exclusive ownership of the object it points to. Only one `unique_ptr` can own a particular object at any given time. ==When the `unique_ptr` is destroyed, the owned object is also destroyed.== It cannot be copied, but it can be moved, transferring ownership.
>2. **`std::shared_ptr`**: Implements shared ownership. Multiple `shared_ptr`s can point to the same object. A reference counter tracks how many `shared_ptr`s currently own the object. ==The object is deleted only when the last `shared_ptr` owning it is destroyed or reset.==
>3. **`std::weak_ptr`**: A non-owning smart pointer that works in conjunction with `std::shared_ptr`. It does not affect the reference count of the object it points to. It's used to break circular references between `shared_ptr`s, which can lead to memory leaks if not handled. A `weak_ptr` can be converted to a `shared_ptr` to safely access the object; if the object has already been deleted, the conversion will result in a null `shared_ptr`.

# **Related Concepts:**

---
- **RAII (Resource Acquisition Is Initialization):** This is the fundamental programming idiom that smart pointers leverage. RAII states that resource acquisition should happen in the constructor of an object and resource release in its destructor. Smart pointers are prime examples of RAII in action, managing memory (a resource) automatically.
- **Raw Pointers:** These are the traditional C++ pointers. Smart pointers abstract away the complexities and dangers of raw pointers by automating memory management. While raw pointers offer direct memory access and flexibility, they come with the burden of manual deallocation. Smart pointers aim to provide the benefits of pointers without their pitfalls.
- **Memory Leaks:** This occurs when dynamically allocated memory is no longer reachable by the program but has not been deallocated. Smart pointers directly combat memory leaks by ensuring that owned memory is automatically released when no longer needed.
- **Dangling Pointers:** A pointer that points to a memory location that has been deallocated. Using a dangling pointer leads to undefined behavior. `std::unique_ptr` inherently prevents dangling pointers by enforcing single ownership, while `std::weak_ptr` helps detect if the object a `shared_ptr` pointed to has been deallocated.
- **[[JavaScript Garbage Collection]]:** An automatic memory management system that identifies and reclaims memory that is no longer in use. While smart pointers offer automated memory management, they are distinct from garbage collection. Smart pointers use deterministic destruction (RAII), meaning memory is released precisely when the smart pointer goes out of scope. Garbage collection is typically non-deterministic and can introduce pauses in program execution for collection cycles. C++ does not have built-in garbage collection; smart pointers provide a C++ idiomatic way to achieve similar memory safety benefits.

# **Examples:**

---
```cpp

# include <iostream>

# include <memory> // Required for unique_ptr, shared_ptr, weak_ptr

# include <vector>

class MyClass {
public:
 int value;
 MyClass(int val) : value(val) {
 std::cout << "MyClass constructor called for value: " << value << std::endl;
 }
 ~MyClass {
 std::cout << "MyClass destructor called for value: " << value << std::endl;
 }
 void doSomething {
 std::cout << "Doing something with value: " << value << std::endl;
 }
};

void processUniquePtr(std::unique_ptr<MyClass> ptr) {
 // This function takes ownership of the unique_ptr.
 // When 'ptr' goes out of scope in this function, the MyClass object will be destroyed.
 if (ptr) {
 ptr->doSomething;
 }
 std::cout << "Exiting processUniquePtr function." << std::endl;
} // 'ptr' is destroyed here, and thus the MyClass object it owns is also destroyed.

int main {
 //
---
std::unique_ptr Example
---
std::cout << "
---
std::unique_ptr Example
---
" << std::endl;

 // Creating a unique_ptr. It exclusively owns the MyClass object.
 std::unique_ptr<MyClass> u_ptr1 = std::make_unique<MyClass>(10);
 // You can access members using -> or *.
 u_ptr1->doSomething;
 std::cout << "Value via u_ptr1: " << (*u_ptr1).value << std::endl;

 // Trying to copy a unique_ptr will result in a compile-time error.
 // std::unique_ptr<MyClass> u_ptr2 = u_ptr1; // ERROR: Call to implicitly-deleted copy constructor

 // Moving ownership from u_ptr1 to u_ptr2. u_ptr1 becomes null.
 std::unique_ptr<MyClass> u_ptr2 = std::move(u_ptr1);
 if (u_ptr1 == nullptr) {
 std::cout << "u_ptr1 is now null after move." << std::endl;
 }
 u_ptr2->doSomething;

 // Passing unique_ptr by value to a function transfers ownership.
 // When processUniquePtr returns, the object owned by 'temp_ptr' will be destroyed.
 std::unique_ptr<MyClass> temp_ptr = std::make_unique<MyClass>(20);
 processUniquePtr(std::move(temp_ptr));
 if (temp_ptr == nullptr) {
 std::cout << "temp_ptr is null after being moved to function." << std::endl;
 }
 std::cout << std::endl;

 //
---
std::shared_ptr Example
---
std::cout << "
---
std::shared_ptr Example
---
" << std::endl;

 // Create a shared_ptr. The reference count is 1.
 std::shared_ptr<MyClass> s_ptr1 = std::make_shared<MyClass>(30);
 std::cout << "s_ptr1 reference count: " << s_ptr1.use_count << std::endl;

 // Copying a shared_ptr increases the reference count.
 std::shared_ptr<MyClass> s_ptr2 = s_ptr1;
 std::cout << "s_ptr1 reference count: " << s_ptr1.use_count << std::endl;
 std::cout << "s_ptr2 reference count: " << s_ptr2.use_count << std::endl;

 // Another copy, reference count increases again.
 std::shared_ptr<MyClass> s_ptr3 = s_ptr1;
 std::cout << "s_ptr1 reference count: " << s_ptr1.use_count << std::endl;

 s_ptr1->doSomething;

 // When s_ptr2 goes out of scope (or is reset), the reference count decreases.
 s_ptr2.reset; // Explicitly reset s_ptr2, releasing its ownership.
 std::cout << "s_ptr1 reference count after s_ptr2 reset: " << s_ptr1.use_count << std::endl;

 // The MyClass object will only be destroyed when the last shared_ptr (s_ptr3)
 // owning it goes out of scope or is reset.
 // In this case, s_ptr3 will be destroyed when main finishes.
 std::cout << std::endl;

 //
---
std::weak_ptr Example
---
std::cout << "
---
std::weak_ptr Example
---
" << std::endl;

 std::shared_ptr<MyClass> shared_res = std::make_shared<MyClass>(40);
 std::weak_ptr<MyClass> weak_res = shared_res; // Create a weak_ptr from a shared_ptr.
 // This does NOT increase the reference count.

 std::cout << "shared_res reference count: " << shared_res.use_count << std::endl;

 // Lock the weak_ptr to get a shared_ptr and access the object.
 if (auto locked_ptr = weak_res.lock) {
 locked_ptr->doSomething;
 std::cout << "Object is still alive, accessed via weak_ptr." << std::endl;
 } else {
 std::cout << "Object no longer exists." << std::endl;
 }

 shared_res.reset; // Destroy the object by resetting the shared_ptr.
 // Now the reference count for the object becomes 0.

 // Try to access the object again via the weak_ptr.
 if (auto locked_ptr = weak_res.lock) {
 locked_ptr->doSomething;
 std::cout << "Object is still alive (this shouldn't happen after reset)." << std::endl;
 } else {
 std::cout << "Object no longer exists, as expected after shared_res reset." << std::endl;
 }

 std::cout << std::endl;

 // Example with a vector of unique_ptr
 std::cout << "
---

Vector of unique_ptr Example
---
" << std::endl;
 std::vector<std::unique_ptr<MyClass>> my_objects;
 my_objects.push_back(std::make_unique<MyClass>(50));
 my_objects.push_back(std::make_unique<MyClass>(60));
 // When the vector goes out of scope, each unique_ptr in it will be destroyed,
 // and consequently, the MyClass objects they own will be destroyed.
 std::cout << "Objects in vector will be destroyed when vector goes out of scope." << std::endl;

 return 0;
} // All smart pointers (s_ptr3, my_objects elements) go out of scope here,
 // triggering destructors for the owned MyClass objects.
```

# **Flashcards:**

---
What is the primary problem smart pointers solve?;; Automatic memory management and prevention of memory leaks and dangling pointers in C++.

What are the three main types of smart pointers in C++ and their ownership models?;; `std::unique_ptr` (exclusive ownership), `std::shared_ptr` (shared ownership with reference counting), `std::weak_ptr` (non-owning, used with `shared_ptr` to break circular references).

How do smart pointers relate to RAII?;; Smart pointers are a prime example of RAII (Resource Acquisition Is Initialization), where resources (like memory) are acquired in the constructor and released in the destructor, ensuring automatic cleanup.