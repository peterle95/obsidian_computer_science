---
memory: to_finish
tags:
  - learned
language:
  - C++
review-date:
last-reviewed: 2025-10-24
scheda: done
visit-count: 4
confidence-level: 3
consecutive-correct: 4
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
# Purpose/Why:
---

In C++, sequence containers are a fundamental part of the Standard Template Library (STL) that address the need to ==store and manage collections of elements in a linear order.== The primary problem they solve is the ==flexible and efficient storage of ordered data, offering advantages over traditional C-style arrays, such as dynamic resizing and integrated memory management.== Their importance in C++ lies in providing a set of reusable and optimized data structures that empower developers to write cleaner, more efficient, and more expressive code for a wide range of applications, from simple data storage to complex algorithmic problems.

# Core Explanation:
---

C++ sequence containers are class templates that manage a collection of objects of a certain type <mark style="background: #D2B3FFA6;">in a strict linear sequence</mark>. The key characteristic of sequence containers is that the <mark style="background: #D2B3FFA6;">position of an element is determined by the order of its insertion and is not dependent on its value</mark>. They allow for sequential access to elements and provide a common interface for interacting with the stored data.

The C++ Standard Library provides five main sequence containers:

- **[Vector](https://cplusplus.com/reference/vector/vector/)**: A dynamic <u>array that can grow and shrink in size.</u> It provides ==fast random access to elements (O(1))==. Insertions and deletions at the end are efficient (amortized O(1)), but insertions and deletions in the middle or at the beginning are slow (O(n)) because they require shifting subsequent elements.
    
- **[Deque](https://cplusplus.com/reference/deque/deque/)** (double-ended queue): ==Similar to a vector, but with the added efficiency of adding and removing elements from both the front and the back in amortized constant time== (O(1)). Unlike vectors, ==deques are not guaranteed to have their elements in a single contiguous block of memory.==
    
- **[List](https://cplusplus.com/reference/list/list/)**: A <mark style="background: #BBFABBA6;">doubly-linked list</mark>, where <mark style="background: #BBFABBA6;">each element contains a pointer to the previous and the next element</mark> .This structure allows for efficient insertion and deletion of elements anywhere in the list (O(1)) once the position is known. However, <mark style="background: #BBFABBA6;">it does not support fast random access; </mark><mark style="background: #BBFABBA6;">accessing an element by its index requires traversing the list from the beginning or end (O(n)).</mark>
    
- **[Forward_list](https://cplusplus.com/reference/forward_list/forward_list/)**: A <mark style="background: #ABF7F7A6;">singly-linked list,</mark> where <mark style="background: #ABF7F7A6;">each element only points to the next element</mark>. It is more memory-efficient than std::list but can only be iterated in one direction. It also provides constant time insertions and deletions.
    
- **[Array](https://cplusplus.com/reference/array/array/)**: A <mark style="background: #ADCCFFA6;">fixed-size container that wraps a C-style array. It offers the performance benefits of a raw array with the advantages of a standard container, such as bounds checking</mark> (with the at() member function) and iterators. Its size must be known at compile-time.

# Related Concepts:
---

- **Associative Containers**: In contrast to sequence containers, the position of elements in associative containers depends on their key, not the insertion order. They are designed for efficient retrieval of elements based on their key, typically with logarithmic time complexity (O(log n)). Examples include std::set, std::map, std::multiset, and std::multimap.
    
- **Unordered Associative Containers**: These are similar to associative containers in that they use keys to store and access elements. However, they do not maintain a sorted order and instead use hash tables for storage, which allows for average constant-time complexity (O(1)) for insertions, deletions, and lookups. Examples include std::unordered_set, std::unordered_map, std::unordered_multiset, and std::unordered_multimap.
    
- **Container Adapters**: These are not full container classes themselves but are wrappers around other container types (like std::vector, std::deque, or std::list) to provide a specific interface.Examples are std::stack (LIFO), std::queue (FIFO), and std::priority_queue.
    
- **[[C++ Iterators]]**: Iterators are objects that act like pointers and are used to traverse the elements of a container. All standard containers provide iterators to access their elements. Sequence containers provide at least forward iterators, while std::list and std::deque provide bidirectional iterators, and std::vector and std::array provide random access iterators.
    

# Examples:
---
```cpp
#include <iostream>
#include <vector>
#include <list>
#include <deque>
#include <array>
#include <forward_list>
#include <string>
#include <algorithm>

int main() {
    // --- std::vector ---
    // A dynamic array that is resizeable. Good for fast random access. [14, 23]
    std::vector<std::string> my_vector;

    // Add elements to the end of the vector. [18]
    my_vector.push_back("Apple");
    my_vector.push_back("Banana");
    my_vector.push_back("Cherry");

    // Access elements by index (fast). [20]
    std::cout << "Vector element at index 1: " << my_vector[1] << std::endl;

    // Iterate through the vector. [14]
    std::cout << "Vector elements: ";
    for (const auto& fruit : my_vector) {
        std::cout << fruit << " ";
    }
    std::cout << std::endl << std::endl;


    // --- std::list ---
    // A doubly-linked list. Good for frequent insertions and deletions in the middle. [16]
    std::list<int> my_list;

    // Add elements to the front and back. [13]
    my_list.push_front(10);
    my_list.push_back(20);
    my_list.push_back(30);

    // Find an element and insert before it. [15]
    auto it = std::find(my_list.begin(), my_list.end(), 20);
    if (it != my_list.end()) {
        my_list.insert(it, 15);
    }

    // Iterate through the list.
    std::cout << "List elements: ";
    for (int num : my_list) {
        std::cout << num << " ";
    }
    std::cout << std::endl << std::endl;


    // --- std::deque ---
    // A double-ended queue. Good for frequent insertions and deletions at both ends. [6, 25]
    std::deque<char> my_deque;

    // Add elements to the front and back. [19, 22]
    my_deque.push_front('B');
    my_deque.push_back('C');
    my_deque.push_front('A');
    my_deque.push_back('D');

    // Access elements by index. [22, 25]
    std::cout << "Deque element at index 2: " << my_deque[2] << std::endl;

    // Remove elements from the front and back. [25]
    my_deque.pop_front();
    my_deque.pop_back();

    // Iterate through the deque.
    std::cout << "Deque elements: ";
    for (char c : my_deque) {
        std::cout << c << " ";
    }
    std::cout << std::endl << std::endl;


    // --- std::array ---
    // A fixed-size array wrapper. Good for compile-time known sizes with stack allocation.
    std::array<double, 4> my_array = {1.1, 2.2, 3.3, 4.4};

    // Access elements by index (fast, like vector).
    std::cout << "Array element at index 2: " << my_array[2] << std::endl;

    // Get size (compile-time constant).
    std::cout << "Array size: " << my_array.size() << std::endl;

    // Iterate through the array.
    std::cout << "Array elements: ";
    for (const auto& value : my_array) {
        std::cout << value << " ";
    }
    std::cout << std::endl << std::endl;


    // --- std::forward_list ---
    // A singly-linked list. Good for memory efficiency when only forward iteration is needed.
    std::forward_list<std::string> my_forward_list;

    // Add elements to the front (no push_back available).
    my_forward_list.push_front("Third");
    my_forward_list.push_front("Second");
    my_forward_list.push_front("First");

    // Insert after a specific position.
    auto fl_it = my_forward_list.begin();
    std::advance(fl_it, 1); // Move to "Second"
    my_forward_list.insert_after(fl_it, "Between");

    // Remove elements after a specific position.
    fl_it = my_forward_list.begin();
    my_forward_list.erase_after(fl_it); // Remove "Second"

    // Iterate through the forward_list.
    std::cout << "Forward_list elements: ";
    for (const auto& item : my_forward_list) {
        std::cout << item << " ";
    }
    std::cout << std::endl;

    return 0;
}
```

# Flashcards:
---

What is the primary characteristic of a C++ sequence container?;;Elements are stored in a strict linear sequence, and their position depends on the order of insertion, not their value.

Which sequence container is generally the best default choice and why?;;std::vector, because it offers fast random access (O(1)) and efficient additions/removals at the end (amortized O(1)).

When would you prefer to use a std::list over a std::vector?;;When you need to perform frequent insertions and deletions in the middle of the container, as std::list can do this in constant time (O(1)).

What is the main advantage of std::deque over std::vector?;;std::deque allows for efficient insertions and deletions at both the beginning and the end of the container (amortized O(1)).

What is the fundamental difference between sequence containers and associative containers?;;In sequence containers, elements are ordered by their position of insertion, while in associative containers, elements are ordered based on their key.