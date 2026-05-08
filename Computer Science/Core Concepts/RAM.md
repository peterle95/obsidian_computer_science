---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date: ""
last-reviewed: 2025-10-03
scheda: done
visit-count: 3
confidence-level: 1.5
consecutive-correct: 2
last-struggle-date: 2025-07-06
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
**Random Access Memory (RAM)** is a form of computer memory that can be read from and written to in any random order. Unlike storage devices like hard drives or SSDs, RAM is a *volatile* memory, meaning it requires power to maintain the stored information. If the power is removed, all data stored in RAM is lost.

RAM serves as a computer's **primary memory** or **main memory**. It is where the operating system, actively running applications, and data currently in use are temporarily stored. Its purpose is to provide extremely fast access to data for the Central Processing Unit (CPU), significantly faster than accessing data from secondary storage. This high speed is crucial for the smooth and efficient operation of a computer, as the CPU constantly needs to access and process data.

# **Related Concepts:**

---
* **Volatile Memory:** Memory that loses its contents when power is turned off (e.g., RAM).
* **Non-Volatile Memory:** Memory that retains its contents even when power is turned off (e.g., ROM, SSDs, HDDs, flash drives).
* **CPU (Central Processing Unit):** The "brain" of the computer that processes instructions and data. RAM is critical because it provides the CPU with quick access to the data it needs.
* **Cache Memory:** A smaller, extremely fast type of volatile memory located closer to the CPU (often on the CPU chip itself or directly adjacent). It stores copies of data from frequently used main memory locations, allowing the CPU to access it even faster than RAM. Cache memory operates in multiple levels (L1, L2, L3), with L1 being the fastest and closest to the CPU.
* **Virtual Memory:** A memory management technique used by operating systems. When RAM is full, the OS can temporarily move data from RAM to a designated space on a slower storage device (like an SSD or HDD, called the "swap file" or "paging file"). This creates the illusion of having more RAM than physically present, but accessing data from virtual memory is significantly slower than from physical RAM.
* **Memory Modules (DIMM/SO-DIMM):** Physical circuit boards containing RAM chips that are inserted into slots on a motherboard. DIMMs (Dual In-line Memory Modules) are typically used in desktop computers and servers, while SO-DIMMs (Small Outline DIMM) are used in laptops and smaller devices.
* **DRAM (Dynamic RAM):** The most common type of RAM used for main memory. It requires constant refreshing to maintain data.
* **SRAM (Static RAM):** Faster and more expensive than DRAM, SRAM is primarily used for cache memory because it does not need to be refreshed.
* **DDR (Double Data Rate) SDRAM:** The current generation of RAM technology (e.g., DDR4, DDR5). Each successive generation offers higher speeds, lower power consumption, and increased bandwidth.
* **Memory Speed (MHz/MT/s):** Refers to how quickly the RAM can transfer data. Measured in Megahertz (MHz) or Mega Transfers per second (MT/s).
* **Latency (CL/CAS Latency):** The delay between when a command is issued to the RAM and when the data is actually available. Lower latency is better.
* **Bandwidth:** The amount of data that can be transferred to and from memory per unit of time. Higher bandwidth is generally better for performance.

# **Examples:**

---
```cpp

---
EXTENSIVE EXPLANATIONS AS COMMENTS
---
# include <iostream> // For standard input/output operations (e.g., std::cout, std::endl)

# include <vector> // For using std::vector, a dynamic array that stores elements in RAM

# include <chrono> // For measuring time, used to see how long large allocations take

# include <string> // Not strictly needed for this example, but often useful for text manipulation

// This example demonstrates how data is temporarily stored and accessed in RAM
// when a C++ program is executed. It highlights the concept of volatile memory
// and how actively used data resides in RAM for fast CPU access.

int main {
 std::cout << "
---

RAM Usage Demonstration
---
" << std::endl;

 // 1. Variables and basic data types are stored in RAM
 // When you declare variables like these, the compiler allocates space for them
 // on the program's stack (which is a region within RAM).
 // These variables hold the actual values and their memory addresses point to locations in RAM.
 int anInteger = 42; // An integer variable (typically 4 bytes)
 double aDouble = 3.14159; // A double-precision floating-point variable (typically 8 bytes)
 bool aBoolean = true; // A boolean variable (typically 1 byte, but can vary)
 char aChar = 'A'; // A character variable (typically 1 byte)

 std::cout << "Variables declared and stored in RAM:" << std::endl;
 // Using '&' operator to get the memory address of the variable.
 // These addresses are pointers to locations within your computer's RAM.
 std::cout << " anInteger: " << anInteger << " (Address: " << &anInteger << ")" << std::endl;
 std::cout << " aDouble: " << aDouble << " (Address: " << &aDouble << ")" << std::endl;
 // For bool and char pointers, casting to (void*) is good practice for printing.
 std::cout << " aBoolean: " << (aBoolean ? "true" : "false") << " (Address: " << (void*)&aBoolean << ")" << std::endl;
 std::cout << " aChar: " << aChar << " (Address: " << (void*)&aChar << ")" << std::endl;
 std::cout << std::endl;

 // 2. Dynamic memory allocation (Heap memory) also uses RAM
 // The 'new' operator requests a block of memory from the system's heap.
 // The heap is another region within RAM used for dynamic allocations,
 // where the size and lifetime of the memory are managed by the programmer.
 int* dynamicInteger = new int; // Request space for one integer on the heap.
 // 'dynamicInteger' itself is a pointer variable stored on the stack,
 // but the memory it points to is on the heap.
 *dynamicInteger = 100; // Store a value in the dynamically allocated memory location.
 std::cout << "Dynamically allocated integer in RAM (Heap):" << std::endl;
 std::cout << " *dynamicInteger: " << *dynamicInteger << " (Address: " << dynamicInteger << ")" << std::endl;
 // IMPORTANT: For every 'new', there should be a corresponding 'delete'.
 // Failing to 'delete' dynamically allocated memory leads to "memory leaks,"
 // where the program consumes RAM that it no longer uses, but never releases.
 delete dynamicInteger; // Deallocate the memory block pointed to by dynamicInteger.
 dynamicInteger = nullptr; // Good practice to set the pointer to nullptr after deletion
 // to prevent "dangling pointers" (pointers that point to freed memory).
 std::cout << " (Dynamic integer deallocated)" << std::endl;
 std::cout << std::endl;

 // 3. Data structures and collections store their elements in RAM
 // Standard Library containers like std::vector manage a contiguous block of memory in RAM.
 // When elements are added, the vector may dynamically resize its underlying array in RAM.
 std::vector<int> numbers;
 numbers.push_back(10); // Adds an element, potentially resizing the vector's internal array in RAM.
 numbers.push_back(20);
 numbers.push_back(30);

 std::cout << "Vector elements stored in RAM:" << std::endl;
 // Iterate through the vector and print the values and their addresses in RAM.
 for (size_t i = 0; i < numbers.size; ++i) {
 std::cout << " numbers[" << i << "]: " << numbers[i] << " (Address: " << &numbers[i] << ")" << std::endl;
 }
 std::cout << std::endl;

 // 4. Program instructions themselves are loaded into RAM for execution
 // When you launch this program, the operating system loads its executable code
 // (machine instructions) from the hard drive into RAM. The CPU then fetches
 // these instructions from RAM, decodes them, and executes them. This is how the
 // program "runs."

 // 5. Simulating a large memory allocation to illustrate significant RAM usage
 // This section demonstrates how a program can consume a substantial amount of RAM
 // for its data.
 // 'numElements' defines how many integers we want to store.
 // Each integer typically takes 4 bytes. So, 10 million integers = 40 million bytes.
 // 40,000,000 bytes / (1024 * 1024) bytes/MB = approx. 38 MB.
 const long long numElements = 10000000; // 10 million elements for demonstration
 // If you have a lot of RAM (e.g., 16GB or more) and want to observe a more
 // dramatic impact on system RAM usage, you could uncomment the line below.
 // Be mindful that allocating 100 million integers is about 381 MB.
 // const long long numElements = 100000000;

 std::cout << "Allocating a large vector to simulate significant RAM usage..." << std::endl;
 // Start timing the allocation.
 auto start_time = std::chrono::high_resolution_clock::now;

 // The 'largeVector' is declared. When it's constructed with 'numElements',
 // a contiguous block of memory large enough to hold 'numElements' integers
 // is reserved in RAM. All these integers will reside in RAM until the vector
 // is destroyed or cleared.
 std::vector<int> largeVector(numElements);

 // End timing the allocation.
 auto end_time = std::chrono::high_resolution_clock::now;
 std::chrono::duration<double> duration = end_time - start_time;

 std::cout << "Large vector of " << numElements << " integers allocated." << std::endl;
 std::cout << "Allocation time: " << duration.count << " seconds." << std::endl;
 // Calculate and display the approximate memory consumed by the vector.
 std::cout << "This data occupies approx. " << (numElements * sizeof(int)) / (1024 * 1024.0) << " MB of RAM." << std::endl;

 // This line pauses the program. During this pause, you can open your system's
 // task manager (Windows: Ctrl+Shift+Esc, Linux: 'top' or 'htop', macOS: Activity Monitor)
 // and observe the memory usage of your compiled program. You should see a spike
 // in its memory consumption after the large vector is allocated.
 std::cout << "Press Enter to continue and deallocate the vector..." << std::endl;
 std::cin.ignore; // Waits for a newline character (after you press Enter).

 // When 'largeVector' goes out of scope (as the 'main' function is about to end),
 // its destructor is called, which automatically frees the memory it was using.
 // If you explicitly call 'largeVector.clear', the memory is also freed.
 // This demonstrates the volatile nature of RAM: data is only held as long as
 // the program or system needs it and has power. Once freed, it becomes available
 // for other processes or programs.
 largeVector.clear; // Explicitly deallocate the vector's memory
 std::cout << "Large vector deallocated. RAM usage should decrease." << std::endl;
 std::cout << "
---

End of Demonstration
---
" << std::endl;

 return 0; // Program exits successfully. All remaining program memory is freed by the OS.
}
```

# **Flashcards:**

---
What does "volatile" mean in the context of RAM?;; "Volatile" means that RAM requires continuous power to maintain the stored information. If the power is removed, all data in RAM is lost.

Why is RAM crucial for computer performance, especially in relation to the CPU?;; RAM is the computer's primary memory, providing extremely fast access to actively used data and program instructions for the CPU. This speed is essential because the CPU constantly needs data for processing, and RAM is significantly faster than secondary storage.
