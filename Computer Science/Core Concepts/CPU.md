---
memory: to_finish
tags:
  - mastered
language:
  - Core Concepts
review-date:
last-reviewed: 2025-09-15
scheda: done
visit-count: 2
confidence-level: 1.5
consecutive-correct: 1
last-struggle-date: 2025-09-05
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

The CPU (Central Processing Unit) solves the fundamental problem of ==executing computational instructions in a computer system==. It serves as the "brain" of the computer, responsible for interpreting and executing all software instructions, performing arithmetic and logical operations, and coordinating data flow between different system components. Without a CPU, a computer would be unable to process any information or run any programs. It's the core component that transforms stored program instructions into actual computational work, making it essential for any programmable digital device.

# **Core Explanation:**
---

The CPU is the primary computational component of a computer that ==executes instructions from programs stored in memory==. It operates through a ==continuous cycle called the fetch-decode-execute cycle==: <mark style="background: #BBFABBA6;">first fetching instructions from memory, then decoding what operation needs to be performed, and finally executing that operation.
</mark>
Key characteristics of a CPU include:
- **Clock Speed**: Measured in <mark style="background: #ADCCFFA6;">gigahertz (GHz), determines how many instruction cycles can be completed per second</mark>
- **Cores**: Modern CPUs have multiple processing cores,<mark style="background: #ADCCFFA6;"> allowing parallel execution of instructions</mark>
- **Cache Memory**: High-speed memory built into the CPU for <mark style="background: #ADCCFFA6;">storing frequently accessed data</mark> and instructions
- **Instruction Set Architecture (ISA)**: The set of instructions the CPU can understand and execute
- **Pipeline**: A technique that allows multiple instructions to be processed simultaneously at different stages

The CPU contains several critical components: the Arithmetic Logic Unit (ALU) performs mathematical and logical operations, the Control Unit manages instruction flow and coordinates other components, and registers provide temporary storage for data being processed. The CPU communicates with other system components through buses that carry data, addresses, and control signals.

# **Related Concepts:**
---

**Memory Hierarchy**: CPUs work closely with various types of memory (cache, RAM, storage) in a hierarchical system where faster memory is closer to the CPU but more expensive and limited in size.

**GPU (Graphics Processing Unit)**: Specialized processors designed for parallel processing of graphics operations, differing from CPUs which are optimized for sequential processing and complex instruction handling.

**Instruction Set Architecture (ISA)**: The interface between hardware and software that defines the CPU's capabilities, including instruction formats, addressing modes, and data types (examples: x86, ARM, RISC-V).

**Process and Thread Management**: CPUs handle multiple processes and threads through scheduling algorithms, context switching, and time-sharing mechanisms.

**Computer Architecture**: The overall design and organization of computer systems, where the CPU is the central component that influences and is influenced by memory systems, I/O devices, and interconnects.

**Assembly Language**: Low-level programming language that directly corresponds to CPU instructions, providing a human-readable representation of machine code.

# **Examples:**
---

```assembly
; Simple Assembly Example - Adding Two Numbers
; This demonstrates how CPU instructions work at the lowest level

section .data
 num1 dd 10 ; Define first number (32-bit integer)
 num2 dd 20 ; Define second number (32-bit integer)
 result dd 0 ; Reserve space for the result

section .text
 global _start

_start:
 ; Load first number into EAX register
 ; CPU fetches this instruction from memory
 mov eax, [num1] ; EAX = 10 (load operation)

 ; Add second number to EAX
 ; CPU's ALU performs the addition operation
 add eax, [num2] ; EAX = EAX + 20 = 30 (arithmetic operation)

 ; Store result back to memory
 ; CPU writes the result from register to memory
 mov [result], eax ; result = 30 (store operation)

 ; Exit program
 ; CPU executes system call to terminate program
 mov eax, 1 ; System call number for exit
 mov ebx, 0 ; Exit status
 int 0x80 ; Interrupt to invoke system call
```

```c
// C Code Example - CPU Performance Monitoring
// This shows how to measure CPU performance characteristics

# include <stdio.h>

# include <time.h>

# include <unistd.h>

int main {
 clock_t start, end;
 double cpu_time_used;

 // Record start time - CPU clock cycles
 start = clock;

 // Simulate CPU-intensive work
 // This loop keeps the CPU busy performing calculations
 long long sum = 0;
 for (int i = 0; i < 1000000; i++) {
 sum += i * i; // CPU performs multiplication and addition
 }

 // Record end time
 end = clock;

 // Calculate CPU time used
 // CLOCKS_PER_SEC is CPU clock frequency related
 cpu_time_used = ((double) (end - start)) / CLOCKS_PER_SEC;

 printf("Sum calculated: %lld\n", sum);
 printf("CPU time used: %f seconds\n", cpu_time_used);

 // Get CPU information (system dependent)
 printf("CPU clock ticks per second: %ld\n", CLOCKS_PER_SEC);

 return 0;
}
```

```python

# Python Example - Multi-core CPU Utilization

# Demonstrates how modern CPUs handle parallel processing

import multiprocessing
import time
import os

def cpu_intensive_task(n):
 """
 CPU-intensive function that calculates prime numbers
 This function will utilize one CPU core fully
 """
 def is_prime(num):
 if num < 2:
 return False

# CPU performs many division operations
 for i in range(2, int(num ** 0.5) + 1):
 if num % i == 0:

# Modulo operation uses CPU's ALU
 return False
 return True

# Find all prime numbers up to n
 primes = [i for i in range(2, n) if is_prime(i)]
 return len(primes)

if __name__ == "__main__":

# Get number of CPU cores available
 num_cores = multiprocessing.cpu_count
 print(f"Number of CPU cores: {num_cores}")

# Single-core execution
 start_time = time.time
 result_single = cpu_intensive_task(10000)
 single_core_time = time.time - start_time

# Multi-core execution using process pool

# This distributes work across multiple CPU cores
 start_time = time.time
 with multiprocessing.Pool(processes=num_cores) as pool:

# Divide work among CPU cores
 tasks = [2500, 2500, 2500, 2500]

# Split the work
 results = pool.map(cpu_intensive_task, tasks)

 multi_core_time = time.time - start_time
 result_multi = sum(results)

 print(f"Single-core result: {result_single}, Time: {single_core_time:.2f}s")
 print(f"Multi-core result: {result_multi}, Time: {multi_core_time:.2f}s")
 print(f"Speedup: {single_core_time/multi_core_time:.2f}x")
```

# **Flashcards:**

---
What is the fetch-decode-execute cycle in a CPU?;; The fundamental operating cycle where the CPU: (1) Fetches instructions from memory, (2) Decodes what operation needs to be performed, (3) Executes the operation using the ALU and other components

What are the main components inside a CPU and their functions?;; ALU (Arithmetic Logic Unit) performs mathematical and logical operations, Control Unit manages instruction flow and coordinates components, Registers provide temporary high-speed storage for data being processed

How do multiple CPU cores improve performance compared to a single core?;; Multiple cores allow parallel processing of instructions, enabling the CPU to execute multiple threads or processes simultaneously, potentially providing significant speedup for multi-threaded applications and multitasking scenarios