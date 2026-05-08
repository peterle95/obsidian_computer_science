---
memory: to_finish
tags:
  - learning
language:
  - C++
review-date: 2025-11-25
last-reviewed: 2025-10-18
scheda: done
visit-count: 2
confidence-level: 1.5
consecutive-correct: 1
last-struggle-date: 2025-10-07
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
Unordered associative containers <mark style="background: #ABF7F7A6;">solve the problem of needing extremely fast access to data based on a key, where the order of elements is not important</mark>. Their <mark style="background: #ABF7F7A6;">primary application</mark> is in scenarios where <mark style="background: #ABF7F7A6;">performance of lookups, insertions, and deletions is paramount.</mark> While o<mark style="background: #ABF7F7A6;">rdered associative containers provide logarithmic time complexity (O(log n)), unordered containers offer average constant time complexity (O(1)). This makes them critically important for performance-sensitive applications like implementing caches, symbol tables in compilers, or counting the frequency of items in a large dataset</mark>. In C++, they provide a standardized, high-performance hash table implementation, saving developers from building their own and ensuring efficiency and correctness.

# Core Explanation:

---

C++ unordered associative containers are a <mark style="background: #D2B3FFA6;">set of class templates</mark> in the Standard Template Library <mark style="background: #D2B3FFA6;">(STL) that manage a collection of objects using keys, but without maintaining a sorted order.</mark> Their defining characteristic is <mark style="background: #D2B3FFA6;">their performance: search, insertion, and deletion operations have an average time complexity of O(1).</mark>

These containers ==work by using a **hash table** as their underlying data structure==. ==When an element is inserted, a **hash function** is applied to its key to compute a hash value. This hash value is used to map the key to a specific "bucket" within an array. All elements that hash to the same bucket are stored together (often in a linked list). To find an element, the container just has to hash the key to find the correct bucket and then search only within that small list of elements. The worst-case time complexity can degrade to O(n) if many elements hash to the same bucket (a "hash collision"), but this is rare with a good hash function.==

The four main unordered associative containers are:

- **Unordered_set**: Stores a <mark style="background: #BBFABBA6;">collection of unique keys</mark>.

- **Unordered_map**: Stores <mark style="background: #BBFABBA6;">key-value pairs with unique keys.</mark>

- **Unordered_multiset**: Similar to std::unordered_set, but <mark style="background: #BBFABBA6;">allows duplicate keys.</mark>

- **Unordered_multimap**: Similar to std::unordered_map, but <mark style="background: #BBFABBA6;">allows for multiple elements to have the same key.</mark>

# Related Concepts:

---
- **Associative Containers (std::map, std::set)**: These are the ordered counterparts. They use a self-balancing binary search tree (like a red-black tree) instead of a hash table. The key difference is the trade-off: associative containers maintain a sorted order and guarantee O(log n) performance, while unordered containers sacrifice order for faster average O(1) performance.

- **Hash Functions**: This is the core mechanism that powers unordered containers. A hash function is a function that takes a key and computes a single integer value (a hash). A good hash function produces a uniform distribution of hash values to minimize collisions. C++ provides a default std::hash for primitive types and some standard library types.

- **Load Factor**: This is a measure of how full the hash table is, calculated as (number of elements) / (number of buckets). As the load factor increases, so does the probability of collisions and performance degradation. Unordered containers automatically resize and rehash their elements into a larger number of buckets when the load factor exceeds a certain threshold.

- **Sequence Containers (std::vector, std::list)**: These differ fundamentally in their access method. Sequence containers store elements by their position in a sequence, and you access them using an index or by iterating. Unordered containers store elements by key, providing fast access directly to the element you need, regardless of its position.

# Examples:

---
```cpp

# include <iostream>
# include <unordered_map>
# include <unordered_set>
# include <string>

int main {
 //
---
std::unordered_map
---
// An unordered_map stores key-value pairs with unique keys.
 // It's extremely fast for lookups, insertions, and deletions.
 // Let's use it to count word frequencies.
 std::unordered_map<std::string, int> word_counts;

 // Insert some words. The map will count occurrences.
 // Accessing a key with that doesn't exist will create it.
 word_counts["apple"]++;
 word_counts["banana"]++;
 word_counts["apple"]++;

 // Check the count for a specific word.
 std::cout << "The count of 'apple' is: " << word_counts["apple"] << std::endl;
 std::cout << "The count of 'cherry' is: " << word_counts["cherry"] << std::endl; // Will be 0 as it was just created.

 // Iterate through the map. Note that the order is not guaranteed.
 std::cout << "\nWord counts:" << std::endl;
 for (const auto& pair : word_counts) {
 std::cout << pair.first << ": " << pair.second << std::endl;
 }
 std::cout << std::endl;

 //
---
std::unordered_set
---
// An unordered_set stores unique elements.
 // It's excellent for quickly checking if an item exists in a collection.
 std::unordered_set<std::string> unique_words;

 // Insert some elements.
 unique_words.insert("hello");
 unique_words.insert("world");
 unique_words.insert("hello"); // This duplicate will be ignored.

 // Check for the presence of an element using find.
 // find returns an iterator to the end of the container if the element is not found.
 if (unique_words.find("world") != unique_words.end) {
 std::cout << "'world' is in the set." << std::endl;
 } else {
 std::cout << "'world' is not in the set." << std::endl;
 }

 if (unique_words.find("goodbye") != unique_words.end) {
 std::cout << "'goodbye' is in the set." << std::endl;
 } else {
 std::cout << "'goodbye' is not in the set." << std::endl;
 }

 // Iterate through the set. Again, the order is not guaranteed.
 std::cout << "\nUnique words in the set:" << std::endl;
 for (const auto& word : unique_words) {
 std::cout << word << " ";
 }
 std::cout << std::endl;

 return 0;
}
```

# Flashcards:

---
What is the primary advantage of unordered associative containers over their ordered counterparts?;;They offer average constant time O(1) complexity for search, insertion, and deletion operations.

What data structure is used internally by unordered associative containers?;;A hash table.

What is the main trade-off for using an unordered container like std::unordered_map instead of std::map?;;You gain significant speed on average but lose the guarantee that elements will be stored and iterated in a sorted order.

What causes the worst-case O(n) performance in an unordered container?;;Hash collisions, where multiple keys are mapped to the same bucket by the hash function.

What is the difference between std::unordered_map and std::unordered_set?;;std::unordered_map stores key-value pairs, while std::unordered_set stores only unique keys.

What two requirements must a custom type meet to be used as a key in an unordered container?;;A specialization for std::hash (a hash function) and an equality comparison operator (operator\==) must be provided for the type.