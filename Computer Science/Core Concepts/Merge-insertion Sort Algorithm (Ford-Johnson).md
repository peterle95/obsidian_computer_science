---
memory: to_finish
tags:
  - will_learn
language:
  - Core Concepts
review-date:
last-reviewed: ""
scheda: done
visit-count: 0
confidence-level: 1
consecutive-correct: 0
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

The Ford-Johnson merge-insert algorithm, also known as the merge-insert sort, solves the fundamental problem of sorting a list of elements in ascending or descending order. Its primary application lies in scenarios where minimizing the number of comparisons is critical, especially for smaller data sets. It's particularly important in theoretical computer science for its near-optimal comparison count, being one of the most efficient comparison sorts in terms of the number of comparisons made, often outperforming quicksort and mergesort for smaller `N`. While not always chosen for general-purpose sorting due to its more complex implementation, its theoretical efficiency makes it a subject of study for understanding the lower bounds of sorting complexity.

# **Core Explanation:**
---

The Ford-Johnson algorithm is a comparison-based sorting algorithm that operates on the principle of divide and conquer, aiming to minimize the total number of comparisons. It combines elements of merging and insertion sort in a specific, optimized way.

**Key Characteristics:**
*   **Comparison-based:** Relies solely on comparing elements to determine their relative order.
*   **Near-optimal comparisons:** Achieves a comparison count very close to the theoretical lower bound for sorting.
*   **Stable (with careful implementation):** Can be implemented to maintain the relative order of equal elements.
*   **Complex Implementation:** More intricate to implement than simpler sorting algorithms.

**How it Works (Steps):**

1.  **Pairing and Sorting within Pairs:** The input list is divided into `n/2` pairs (with one element potentially left as a singleton if `n` is odd). Each pair is then sorted internally (e.g., `(a, b)` becomes `(min(a,b), max(a,b))`). This takes one comparison per pair.

2.  **Recursively Sort the Larger Elements (Main Chain):** The larger element from each sorted pair (and the singleton, if any) forms a new list. This new list of "larger" elements is then recursively sorted using the Ford-Johnson algorithm itself. This creates what is known as the "main chain" or "sorted sequence of larger elements."

3.  **Insert Remaining Elements (Pendulum/Jacobsthal Insertion):** The smaller elements from each pair are now inserted into the sorted main chain. This insertion is done in a specific order to minimize comparisons. The strategy often involves using a "Jacobsthal number" sequence to determine the order of insertion.
    *   The smallest element that has not been inserted yet (which is guaranteed to be the smallest of all elements) is inserted first into the main chain.
    *   Subsequent elements are inserted in groups defined by Jacobsthal numbers. For example, the first `J_k` elements are inserted, then the next `J_{k-1}` elements, and so on. This order ensures that when an element is inserted, the search space for its correct position in the main chain is limited, as elements tend to be inserted near their paired larger elements. Binary search is typically used for insertion into the already sorted main chain.

The efficiency of Ford-Johnson comes from this carefully chosen insertion order, which minimizes the total number of binary search comparisons.

# **Related Concepts:**
---

*   **Comparison Sorting:** A class of sorting algorithms that rely only on comparisons between elements to determine their relative order. Ford-Johnson is a prime example, alongside Merge Sort, Quick Sort, Heap Sort, and Insertion Sort. It differs by optimizing the *number* of comparisons to a near-optimal theoretical limit, unlike some others which prioritize average-case performance or simplicity.
*   **Divide and Conquer:** A general algorithm design paradigm that Ford-Johnson employs. It involves breaking a problem into smaller subproblems, solving those subproblems, and then combining their solutions to solve the original problem. Ford-Johnson applies this by sorting pairs, then recursively sorting larger elements, and finally inserting smaller ones.
*   **Merge Sort:** Another comparison-based, divide-and-conquer algorithm. Merge sort divides the list into halves, sorts them recursively, and then merges the two sorted halves. Ford-Johnson is more complex in its splitting and merging strategy, specifically designed to reduce the *count* of comparisons, whereas Merge Sort is often preferred for its guaranteed O(N log N) time complexity and stability in practice.
*   **Insertion Sort:** A simple sorting algorithm that builds the final sorted array (or list) one item at a time. It iterates through the input elements and at each iteration removes one element from the input, finds the place within the already sorted list, and inserts it there. Ford-Johnson uses an optimized form of insertion (often binary insertion) in its final phase, but the overall structure is much more sophisticated, integrating pairing and recursive sorting first.
*   **Jacobsthal Numbers:** A sequence of integers similar to Fibonacci numbers, defined by `J_n = J_{n-1} + 2*J_{n-2}` with `J_0=0, J_1=1`. These numbers are crucial in determining the optimal insertion order of the smaller elements into the main chain in the Ford-Johnson algorithm. They help group elements for insertion to minimize the number of comparisons required for binary searching their correct positions.

# **Examples:**
---
### Python Example:

```python
import math

def ford_johnson_sort(arr):
    n = len(arr)
    if n <= 1:
        return arr

    # Step 1: Pair up and sort within pairs
    pairs = []
    # Create pairs (val, index) to maintain original index if needed, or just values
    # Here we just sort values directly within pairs
    for i in range(0, n - (n % 2), 2):
        if arr[i] > arr[i+1]:
            pairs.append((arr[i+1], arr[i])) # Store (smaller, larger)
        else:
            pairs.append((arr[i], arr[i+1]))
    
    # Handle the singleton element if n is odd
    singleton = None
    if n % 2 == 1:
        singleton = arr[n-1]

    # Step 2: Recursively sort the larger elements (main chain)
    # Extract all larger elements from the pairs
    main_chain_elements = [p[1] for p in pairs]
    if singleton is not None:
        main_chain_elements.append(singleton) # Add singleton to main chain to be sorted

    # Recursively sort these larger elements
    # For small N, direct sorting might be used, but for a true Ford-Johnson, it's recursive
    if len(main_chain_elements) > 1:
        main_chain_sorted = ford_johnson_sort(main_chain_elements)
    else:
        main_chain_sorted = main_chain_elements

    # Construct the main chain and the remaining 'smaller' elements
    # Map smaller elements to their original larger pairs in the sorted main chain
    # This is a simplification; in a full implementation, you'd track the original larger
    # elements' positions to correctly associate the smaller elements.
    # For this example, we'll just keep a list of the smaller elements to insert.
    smaller_elements_to_insert = [p[0] for p in pairs]

    # Insert the smallest of the smaller elements first (it's the global minimum)
    # This is often the first element of the first pair (if pairs are sorted)
    if not smaller_elements_to_insert: # Handle case with only singleton or no pairs
        return main_chain_sorted
    
    # Find the smallest element among the 'smaller_elements_to_insert'
    # This is usually the smallest element overall in the original array
    min_val_to_insert = float('inf')
    min_val_index = -1
    for i, val in enumerate(smaller_elements_to_insert):
        if val < min_val_to_insert:
            min_val_to_insert = val
            min_val_index = i
            
    # Remove the true minimum from the list of elements to insert
    if min_val_index != -1:
        smaller_elements_to_insert.pop(min_val_index)

    # The main chain now has the global maximum, so place the global minimum
    # This is a simplification. In reality, the smallest of the smaller elements
    # (which is the global minimum) would be inserted at the beginning.
    # Let's assume we insert it at the correct position.
    
    # Start with the main chain (which includes the global max of the smaller elements group)
    # and the very first element (global min).
    # The actual implementation of Ford-Johnson insertion is quite complex,
    # involving Jacobsthal numbers to determine the order of insertion for the 'smaller' elements
    # to minimize comparisons using binary search.
    
    # For simplicity, we will simulate a more straightforward insertion process for the remaining
    # smaller elements, but acknowledge that the true algorithm uses a specific sequence.

    # Initialize the result with the first element (the smallest of all)
    # and then the sorted main chain.
    result = []
    if min_val_to_insert != float('inf'): # If there was a true minimum
        result.append(min_val_to_insert)
    result.extend(main_chain_sorted)
    
    # Step 3: Insert the remaining smaller elements into the sorted result list
    # using binary search.
    # The true Ford-Johnson would use a specific insertion order (Jacobsthal sequence).
    # Here, for demonstration, we'll just insert them one by one.
    # This part of the example deviates from the true optimal comparison count
    # by not using the Jacobsthal sequence, but demonstrates the general idea of insertion.

    for val in smaller_elements_to_insert:
        # Perform binary search to find the insertion point
        low, high = 0, len(result) - 1
        insert_idx = 0
        while low <= high:
            mid = (low + high) // 2
            if result[mid] < val:
                low = mid + 1
            else:
                high = mid - 1
        insert_idx = low
        result.insert(insert_idx, val)

    return result

# Helper function to generate Jacobsthal numbers for insertion order
# J(n) = J(n-1) + 2*J(n-2)
def generate_jacobsthal_sequence(max_val):
    jacobsthal = [0, 1]
    while jacobsthal[-1] < max_val:
        next_val = jacobsthal[-1] + 2 * jacobsthal[-2]
        jacobsthal.append(next_val)
    return jacobsthal[1:] # Exclude J_0=0

# A more accurate (but still simplified) illustration of the insertion step using Jacobsthal numbers
def ford_johnson_sort_with_jacobsthal_insertion(arr):
    n = len(arr)
    if n <= 1:
        return arr

    # Step 1: Pair up and sort within pairs
    pairs_smaller_larger = [] # Stores (smaller, larger) tuples
    main_chain_candidates = [] # Stores the 'larger' elements
    
    for i in range(0, n - (n % 2), 2):
        if arr[i] > arr[i+1]:
            pairs_smaller_larger.append((arr[i+1], arr[i]))
            main_chain_candidates.append(arr[i])
        else:
            pairs_smaller_larger.append((arr[i], arr[i+1]))
            main_chain_candidates.append(arr[i+1])
    
    singleton_val = None
    if n % 2 == 1:
        singleton_val = arr[n-1]
        main_chain_candidates.append(singleton_val)

    # Step 2: Recursively sort the larger elements (main chain)
    main_chain_sorted = ford_johnson_sort_with_jacobsthal_insertion(main_chain_candidates)

    # The result array will be built from the sorted main chain
    # and inserted smaller elements.
    result = main_chain_sorted

    # The smallest element of all is guaranteed to be the first element of the first pair
    # (after sorting pairs). Insert it into the 'result' (which currently only contains main_chain_sorted).
    if pairs_smaller_larger:
        global_min = pairs_smaller_larger[0][0]
        result.insert(0, global_min) # Insert at the beginning
        # We've inserted the first smaller element, remove it from consideration
        elements_to_insert = [p[0] for p in pairs_smaller_larger[1:]]
    else: # Case where there are no pairs, only a singleton or empty list
        elements_to_insert = []

    # Map remaining smaller elements to their associated larger element from the main chain
    # This requires knowing which 'larger' element was paired with each 'smaller' element.
    # For a full implementation, you'd need to track original indices or create more complex structures.
    # For this simplified example, we'll assume `elements_to_insert` are sorted by their original paired larger value
    # or that the insertion order is just based on their values.

    # Generate Jacobsthal numbers for controlled insertion
    jacobsthal_sequence = generate_jacobsthal_sequence(len(elements_to_insert) + 1)
    
    inserted_count = 0
    jacobsthal_idx = 0
    
    while inserted_count < len(elements_to_insert):
        # Determine the number of elements to insert in this block
        current_block_size = jacobsthal_sequence[jacobsthal_idx] - (jacobsthal_sequence[jacobsthal_idx-1] if jacobsthal_idx > 0 else 0)
        
        # Iterate backwards through the elements in the current block
        # This allows us to insert without shifting indices affecting subsequent insertions in the block
        
        # Get indices for the elements to be inserted in this block
        # The elements are taken from the 'remaining smaller elements' list.
        # This part is highly conceptual and needs careful indexing in a real implementation.
        # For simplicity, we will just iterate through the elements_to_insert list
        # and insert them based on the Jacobsthal pattern for their position.
        
        # In a true Ford-Johnson, the elements_to_insert would be grouped according to their
        # proximity to elements in the main chain, and inserted in a specific order
        # dictated by Jacobsthal numbers.
        
        # For this example, let's just insert all remaining smaller elements sequentially
        # using binary search, as implementing the exact Jacobsthal insertion order
        # for a generic example is very complex without index tracking.
        # The true algorithm prioritizes which of the remaining smaller elements to insert next.

        # Simplified Insertion:
        for val in elements_to_insert:
            low, high = 0, len(result) - 1
            insert_idx = 0
            while low <= high:
                mid = (low + high) // 2
                if result[mid] < val:
                    low = mid + 1
                else:
                    high = mid - 1
            insert_idx = low
            result.insert(insert_idx, val)
        
        inserted_count = len(elements_to_insert) # All inserted in this simplified loop
        
        # Increment jacobsthal_idx if we were truly following the sequence
        jacobsthal_idx += 1

    return result

# Example Usage:
my_list = [5, 2, 8, 1, 9, 4, 7, 3, 6]
print(f"Original list: {my_list}")
# Simplified version for demonstration purposes
sorted_list_simple = ford_johnson_sort(my_list.copy())
print(f"Sorted list (simplified Ford-Johnson): {sorted_list_simple}")

# Demonstrating the conceptual Jacobsthal insertion (simplified implementation)
my_list_2 = [5, 2, 8, 1, 9, 4, 7, 3, 6]
print(f"Original list: {my_list_2}")
sorted_list_jacobsthal = ford_johnson_sort_with_jacobsthal_insertion(my_list_2.copy())
print(f"Sorted list (conceptual Jacobsthal Ford-Johnson): {sorted_list_jacobsthal}")

my_list_3 = [3, 1, 4, 1, 5, 9, 2, 6]
print(f"Original list: {my_list_3}")
sorted_list_3 = ford_johnson_sort_with_jacobsthal_insertion(my_list_3.copy())
print(f"Sorted list (conceptual Jacobsthal Ford-Johnson): {sorted_list_3}")
````

### JavaScript Example:

```js
/**
 * Generates a sequence of Jacobsthal numbers up to a given maximum value.
 * Jacobsthal numbers are used to determine the optimal insertion order in Ford-Johnson.
 * J(n) = J(n-1) + 2*J(n-2), with J(0)=0, J(1)=1.
 * @param {number} maxVal - The maximum value the Jacobsthal number should not exceed.
 * @returns {number[]} An array of Jacobsthal numbers (excluding J_0).
 */
function generateJacobsthalSequence(maxVal) {
    const jacobsthal = [0, 1];
    while (jacobsthal[jacobsthal.length - 1] < maxVal) {
        const nextVal = jacobsthal[jacobsthal.length - 1] + 2 * jacobsthal[jacobsthal.length - 2];
        jacobsthal.push(nextVal);
    }
    return jacobsthal.slice(1); // Exclude J_0=0
}

/**
 * Performs a binary search on a sorted array to find the insertion point for a value.
 * @param {number[]} arr - The sorted array to search within.
 * @param {number} val - The value to find the insertion point for.
 * @returns {number} The index where the value should be inserted to maintain sorted order.
 */
function binarySearchInsertionPoint(arr, val) {
    let low = 0;
    let high = arr.length - 1;
    let insertIdx = 0;

    while (low <= high) {
        const mid = Math.floor((low + high) / 2);
        if (arr[mid] < val) {
            low = mid + 1;
        } else {
            high = mid - 1;
        }
    }
    insertIdx = low;
    return insertIdx;
}

/**
 * Implements the Ford-Johnson (Merge-Insert) sorting algorithm.
 * This implementation aims to illustrate the core logic, especially the Jacobsthal-guided insertion.
 * Note: A truly optimal and comparison-count-minimal implementation is significantly more complex
 * due to meticulous tracking of paired elements and precise insertion ranges.
 * This version provides a robust conceptual understanding.
 * @param {number[]} arr - The array of numbers to be sorted.
 * @returns {number[]} The sorted array.
 */
function fordJohnsonSort(arr) {
    const n = arr.length;
    if (n <= 1) {
        return [...arr]; // Return a shallow copy for consistency
    }

    // Step 1: Pair up and sort within pairs
    // Store pairs as { smaller: number, larger: number }
    const pairs = [];
    const mainChainCandidates = []; // Will store the 'larger' elements from each pair

    for (let i = 0; i < n - (n % 2); i += 2) {
        if (arr[i] > arr[i + 1]) {
            pairs.push({ smaller: arr[i + 1], larger: arr[i] });
            mainChainCandidates.push(arr[i]);
        } else {
            pairs.push({ smaller: arr[i], larger: arr[i + 1] });
            mainChainCandidates.push(arr[i + 1]);
        }
    }

    // Handle the singleton element if n is odd
    let singletonVal = null;
    if (n % 2 === 1) {
        singletonVal = arr[n - 1];
        mainChainCandidates.push(singletonVal);
    }

    // Step 2: Recursively sort the larger elements (main chain)
    // The main chain is guaranteed to be sorted after this step.
    const mainChainSorted = fordJohnsonSort(mainChainCandidates);

    // Initialize the result array with the sorted main chain.
    // The global minimum (which is the 'smaller' element of the first pair)
    // will be inserted at the very beginning.
    let result = [];
    let elementsToInsert = []; // List of smaller elements that need to be inserted

    if (pairs.length > 0) {
        // The first 'smaller' element from the first pair is guaranteed to be the global minimum.
        const globalMin = pairs[0].smaller;
        result.push(globalMin);
        
        // Add the rest of the smaller elements to a list for insertion
        // Exclude the first one as it's already handled as globalMin.
        for (let i = 1; i < pairs.length; i++) {
            elementsToInsert.push(pairs[i].smaller);
        }
    }

    // Append the sorted main chain (excluding any element that was the global min,
    // though in this structure, mainChainSorted won't contain globalMin,
    // as it was only from 'larger' elements or singleton)
    result = result.concat(mainChainSorted);

    // Step 3: Insert remaining smaller elements using Jacobsthal-guided insertion.
    // This is the most complex part to implement perfectly.
    // The goal is to insert elements in an order that minimizes comparisons,
    // often by inserting elements close to their paired 'larger' elements in the main chain.

    // Generate Jacobsthal numbers for insertion steps.
    const jacobsthalSequence = generateJacobsthalSequence(elementsToInsert.length + 1);

    // Map smaller elements to their associated larger element from the main chain.
    // This is crucial for the optimal insertion point, but difficult to do generically
    // without tracking original indices or complex objects.
    // For this example, we'll simplify and just have a list of smaller elements to insert.
    // In a full, optimized Ford-Johnson, 'elementsToInsert' would be grouped based on
    // their corresponding 'larger' elements' positions in `mainChainSorted`.

    let currentJacobsthalIdx = 0;
    let insertedCount = 0;
    let currentJacobsthalBlockEnd = 0; // Represents the end index in the elementsToInsert list for current block

    while (insertedCount < elementsToInsert.length) {
        currentJacobsthalIdx++;
        if (currentJacobsthalIdx >= jacobsthalSequence.length) {
            // If we run out of Jacobsthal sequence, insert remaining elements in order
            currentJacbsthalBlockEnd = elementsToInsert.length;
        } else {
            currentJacbsthalBlockEnd = jacobsthalSequence[currentJacobsthalIdx] - 1; // -1 for 0-based indexing
        }

        // Ensure we don't go beyond the actual number of elements available
        currentJacbsthalBlockEnd = Math.min(currentJacbsthalBlockEnd, elementsToInsert.length - 1);
        
        // The Jacobsthal sequence defines which 'blocks' of smaller elements to insert.
        // We iterate backwards from the end of the current block to the start of the previous block.
        // This is to avoid issues with array shifting when inserting.
        const prevJacobsthalBlockEnd = (currentJacobsthalIdx - 1 > 0) ? jacobsthalSequence[currentJacobsthalIdx - 1] - 1 : -1;

        for (let i = currentJacbsthalBlockEnd; i > prevJacobsthalBlockEnd && i >= 0; i--) {
            if (i < elementsToInsert.length) { // Ensure index is valid
                const valToInsert = elementsToInsert[i];
                const insertIdx = binarySearchInsertionPoint(result, valToInsert);
                result.splice(insertIdx, 0, valToInsert);
                insertedCount++;
            }
        }
    }

    return result;
}
```
// --- Example Usage ---
const list1 = [5, 2, 8, 1, 9, 4, 7, 3, 6];
console.log("Original list 1:", list1);
const sortedList1 = fordJohnsonSort(list1);
console.log("Sorted list 1 (Ford-Johnson):", sortedList1); // Expected: [1, 2, 3, 4, 5, 6, 7, 8, 9]

const list2 = [3, 1, 4, 1, 5, 9, 2, 6, 5];
console.log("\nOriginal list 2:", list2);
const sortedList2 = fordJohnsonSort(list2);
console.log("Sorted list 2 (Ford-Johnson):", sortedList2); // Expected: [1, 1, 2, 3, 4, 5, 5, 6, 9]

const list3 = [10, 9, 8, 7, 6, 5, 4, 3, 2, 1];
console.log("\nOriginal list 3:", list3);
const sortedList3 = fordJohnsonSort(list3);
console.log("Sorted list 3 (Ford-Johnson):", sortedList3); // Expected: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

const list4 = [42];
console.log("\nOriginal list 4:", list4);
const sortedList4 = fordJohnsonSort(list4);
console.log("Sorted list 4 (Ford-Johnson):", sortedList4); // Expected: [42]

const list5 = [];
console.log("\nOriginal list 5:", list5);
const sortedList5 = fordJohnsonSort(list5);
console.log("Sorted list 5 (Ford-Johnson):", sortedList5); // Expected: []

# **Flashcards:**
---

What is the primary goal of the Ford-Johnson (merge-insert) algorithm?;; To sort a list of elements while minimizing the number of comparisons made.  
What are the three main steps of the Ford-Johnson algorithm?;; 1. Pair up and sort within pairs. 2. Recursively sort the larger elements (main chain). 3. Insert the smaller elements into the main chain using an optimized order.  
Which mathematical sequence is often used to optimize the insertion order in the Ford-Johnson algorithm?;; Jacobsthal numbers.  
In the Ford-Johnson algorithm, what is the "main chain"?;; It's the sorted sequence of the larger elements from each initial pair, recursively sorted.  
Why is the Ford-Johnson algorithm often not used for general-purpose sorting despite its low comparison count?;; It has a more complex implementation compared to other common sorting algorithms like quicksort or mergesort.  
Is the Ford-Johnson algorithm a comparison-based sort or a non-comparison-based sort?;; It is a comparison-based sorting algorithm.