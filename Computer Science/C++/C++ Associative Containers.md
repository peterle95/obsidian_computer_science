---
memory: to_finish
tags:
  - learned
language:
  - C++
review-date:
last-reviewed: 2025-10-22
scheda: done
visit-count: 4
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-10-01
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

Associative containers in C++ solve the fundamental problem of <mark style="background: #BBFABBA6;">efficient storage and retrieval of elements</mark> ==based on keys==. <mark style="background: #BBFABBA6;">Unlike sequence containers where elements are accessed by their position, associative containers allow for fast lookups (typically logarithmic time complexity)</mark> ==using a key==. Their primary application is to manage collections of data where the value of an element determines its location in the container. This is crucial in computer science for <mark style="background: #BBFABBA6;">implementing data structures like dictionaries, phone books, or any scenario requiring rapid searching, insertion, and deletion of elements based on a unique identifier.</mark> In C++, they provide a powerful and optimized alternative to manually implementing search algorithms on arrays or lists.

# Core Explanation:
---

C++ associative containers are a <mark style="background: #ABF7F7A6;">group of class templates in the Standard Template Library (STL) </mark><mark style="background: #ABF7F7A6;">that store elements in a sorted order, which allows for efficient searching. The key characteristic of these containers is that the elements are ordered based on a comparison function, not by the order of insertion.</mark> Internally, they are typically implemented using self-balancing binary search trees (like Red-Black Trees). This structure guarantees that operations like insertion, deletion, and searching have a time complexity of O(log n).

There are four primary associative containers in C++:

- **[Set](https://cplusplus.com/reference/set/set/)**: Stores a collection of <mark style="background: #BBFABBA6;">unique keys in a sorted order</mark>. The <mark style="background: #BBFABBA6;">value of the element is the key itself</mark>. It is <mark style="background: #BBFABBA6;">useful when you need to store a sorted set of unique items</mark>.
    
- **[Map](https://cplusplus.com/reference/map/map/)**: Stores <mark style="background: #BBFABBA6;">key-value pairs where each key is unique and sorted</mark>. It is often referred to as a dictionary and is ideal for <mark style="background: #BBFABBA6;">situations where you need to associate a value with a unique key.</mark>
    
- **[Multiset](https://cplusplus.com/reference/set/multiset/)**: <mark style="background: #BBFABBA6;">Similar to a std::set, but it allows for duplicate keys</mark>. The elements are still stored in a sorted order.
    
- **[Multimap](https://cplusplus.com/reference/map/multimap/)**: <mark style="background: #BBFABBA6;">Similar to a std::map, but it allows for multiple elements to have the same key</mark>. This is useful for one-to-many relationships.
    

# Related Concepts:
---

- **[[C++ Sequence Containers]]**: These containers, such as std::vector and std::list, store elements in a linear sequence. The position of an element is determined by the order of insertion. In contrast, associative containers order elements based on their key.
    
- **[[C++ - Unordered Associative Containers]]**: Introduced in C++11, these containers (std::unordered_set, std::unordered_map, std::unordered_multiset, std::unordered_multimap) also store elements using keys. However, they use hash tables for their internal implementation instead of sorted trees. This results in average constant-time complexity (O(1)) for insertions, deletions, and lookups, but they do not maintain a sorted order of elements.
    
- **[[C++ Iterators]]**: Like all STL containers, associative containers use iterators to access their elements. The iterators for associative containers are bidirectional, meaning they can be traversed in both forward and reverse directions.
    
- **Comparison Functors**: Associative containers rely on a comparison function to determine the order of their elements. By default, they use std::less, which sorts elements in ascending order. This can be customized by providing a different comparison functor.
    

# Examples:
---
```cpp
#include <iostream>
#include <map>
#include <set>
#include <string>

int main() {
    // --- std::map ---
    // A map stores unique key-value pairs, sorted by the key.
    std::map<std::string, int> student_scores;

    // Insert elements into the map.
    student_scores["Alice"] = 95;
    student_scores["Bob"] = 88;
    student_scores.insert(std::make_pair("Charlie", 92));

    // The keys in a map are unique. Attempting to insert a duplicate key will not change the map.
    student_scores["Alice"] = 98; // This will update the value for the key "Alice".

    // Access the value associated with a key.
    std::cout << "Bob's score: " << student_scores["Bob"] << std::endl;

    // Iterate through the map. The elements will be printed in alphabetical order of the keys.
    std::cout << "Student scores:" << std::endl;
    for (const auto& pair : student_scores) {
        std::cout << pair.first << ": " << pair.second << std::endl;
    }
    std::cout << std::endl;


    // --- std::set ---
    // A set stores unique elements, sorted in order.
    std::set<int> unique_numbers;

    // Insert elements into the set.
    unique_numbers.insert(42);
    unique_numbers.insert(17);
    unique_numbers.insert(99);

    // Inserting a duplicate element will be ignored.
    unique_numbers.insert(42);

    // Check if an element exists in the set.
    if (unique_numbers.count(17)) {
        std::cout << "17 is in the set." << std::endl;
    }

    // Iterate through the set. The elements will be printed in ascending order.
    std::cout << "Unique numbers:" << std::endl;
    for (int num : unique_numbers) {
        std::cout << num << " ";
    }
    std::cout << std::endl;

    return 0;
}
```

# Flashcards:
---

What is the primary purpose of C++ associative containers?;;To provide efficient storage and retrieval of elements based on keys, with operations typically in logarithmic time.

What is the main difference between std::map and std::set?;;std::map stores key-value pairs, while std::set stores only keys (where the value is the key itself).

How do associative containers maintain the sorted order of their elements?;;They are typically implemented as self-balancing binary search trees (like Red-Black Trees) and use a comparison function to order the elements.

What is the difference between std::map and std::multimap?;;std::map requires unique keys, whereas std::multimap allows for duplicate keys.

How do associative containers differ from unordered associative containers?;;Associative containers store elements in a sorted order, while unordered associative containers use hash tables and do not maintain a specific order.


