---
memory: to_finish
tags:
 - learned
language:
 - Core Concepts
review-date:
last-reviewed: 2025-08-20
scheda: done
visit-count: 2
confidence-level: 1
consecutive-correct: 1
last-struggle-date: 2025-08-10

---
```dataviewjs
const currentPage = dv.current;
let visitCount = currentPage.file.frontmatter["visit-count"] || 0;
let confidence = currentPage.file.frontmatter["confidence-level"] || 1;
let streak = currentPage.file.frontmatter["consecutive-correct"] || 0;

const container = this.container.createEl('div');
container.style.cssText = `
 background:

# 2a2a2a; border: 1px solid

# 404040; border-radius: 6px;
 padding: 12px; margin: 10px 0; display: inline-block;
`;

// Status display
const status = container.createEl('div');
status.innerHTML = `
 <strong>Learning Progress</strong><br>
 Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
`;
status.style.cssText = 'margin-bottom: 10px; font-size: 13px; color:

# cccccc;';

// Quick feedback buttons
const buttonContainer = container.createEl('div');
['Got it! ✅', 'Struggled ⚠️', 'Failed ❌'].forEach((label, index) => {
 const btn = buttonContainer.createEl('button');
 btn.textContent = label;
 btn.style.cssText = `
 margin-right: 8px; padding: 4px 8px; border: none; border-radius: 3px;
 cursor: pointer; font-size: 11px;
 background: ${['

# 28a745', '

# ffc107', '

# dc3545'][index]};
 color: ${index === 1 ? '

# 000' : '

# fff'};
 `;

 btn.addEventListener('click', async => {
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
 fm["last-reviewed"] = new Date.toISOString.split('T');
 if (index > 0) fm["last-struggle-date"] = new Date.toISOString.split('T');
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
const currentPage = dv.current;
const content = await app.vault.read(app.vault.getAbstractFileByPath(currentPage.file.path));

// Split content into lines
const lines = content.split('\n');
let flashcardLines = ;
let inCodeBlock = false;

// Collect all potential flashcard lines - simplified approach
for (let i = 0; i < lines.length; i++) {
 const line = lines[i];

 // Track code blocks
 if (line.trim.startsWith('```')) {
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
 line.trim.startsWith('const ') ||
 line.trim.startsWith('let ') ||
 line.trim.startsWith('function ') ||
 line.trim.startsWith('return ') ||
 line.trim.startsWith('if (') ||
 line.trim.startsWith('for (') ||
 line.trim.startsWith('while (') ||
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
const flashcards = ;
for (let i = 0; i < filteredLines.length; i++) {
 const line = filteredLines[i];
 try {
 const separatorIndex = line.indexOf(';;');
 if (separatorIndex === -1) continue;

 const front = line.substring(0, separatorIndex).trim;
 const back = line.substring(separatorIndex + 2).trim;

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
 errorMsg.style.cssText = 'background:

# 2a2a2a; padding: 15px; border-radius: 6px; color:

# cccccc;';
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
 background:

# 2a2a2a;
 border: 1px solid

# 404040;
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
title.style.cssText = 'margin: 0; color:

# ffffff;';

const progress = header.createEl('div');
progress.style.cssText = 'color:

# cccccc; font-size: 14px; text-align: right;';

// Card container
const cardContainer = container.createEl('div');
cardContainer.style.cssText = `
 background:

# 1a1a1a;
 border: 2px solid

# 404040;
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
 color:

# ffffff;
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
 background:

# 4a9eff; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; font-weight: 500;
`;

const easyButton = buttonContainer.createEl('button');
easyButton.textContent = 'Easy ✅';
easyButton.style.cssText = `
 background:

# 28a745; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const hardButton = buttonContainer.createEl('button');
hardButton.textContent = 'Hard ❌';
hardButton.style.cssText = `
 background:

# dc3545; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const nextButton = buttonContainer.createEl('button');
nextButton.textContent = 'Next →';
nextButton.style.cssText = `
 background:

# 6c757d; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const prevButton = buttonContainer.createEl('button');
prevButton.textContent = '← Prev';
prevButton.style.cssText = `
 background:

# 6c757d; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const shuffleButton = buttonContainer.createEl('button');
shuffleButton.textContent = '🔀 Shuffle';
shuffleButton.style.cssText = `
 background:

# 17a2b8; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

// Functions
function updateDisplay {
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
 cardContainer.style.borderColor = '

# ffc107';
 cardContainer.style.backgroundColor = '

# 252525';
 } else {
 easyButton.style.display = 'none';
 hardButton.style.display = 'none';
 flipButton.textContent = 'Flip Card';
 cardContainer.style.borderColor = '

# 404040';
 cardContainer.style.backgroundColor = '

# 1a1a1a';
 }

 // Update navigation buttons
 prevButton.style.display = currentCardIndex > 0 ? 'inline-block' : 'none';
 nextButton.textContent = currentCardIndex < flashcards.length - 1 ? 'Next →' : 'Restart';
}

function flipCard {
 showingBack = !showingBack;
 updateDisplay;
}

function nextCard {
 if (currentCardIndex < flashcards.length - 1) {
 currentCardIndex++;
 } else {
 currentCardIndex = 0;
 }
 showingBack = false;
 updateDisplay;
}

function prevCard {
 if (currentCardIndex > 0) {
 currentCardIndex--;
 showingBack = false;
 updateDisplay;
 }
}

function markCorrect {
 if (showingBack) {
 correctCount++;
 totalReviewed++;
 nextCard;
 }
}

function markIncorrect {
 if (showingBack) {
 totalReviewed++;
 nextCard;
 }
}

function shuffleCards {
 for (let i = flashcards.length - 1; i > 0; i--) {
 const j = Math.floor(Math.random * (i + 1));
 [flashcards[i], flashcards[j]] = [flashcards[j], flashcards[i]];
 }
 currentCardIndex = 0;
 showingBack = false;
 correctCount = 0;
 totalReviewed = 0;
 updateDisplay;
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
instructions.style.cssText = 'font-size: 12px; color:

# 888; text-align: center; line-height: 1.4;';
instructions.innerHTML = `
 <strong>Controls:</strong> Click card to flip | Navigation buttons | Easy/Hard to mark
`;

// Initialize
updateDisplay;
```

# **Purpose/Why**:

---
Processes solve the fundamental problem of how an operating system can manage and execute multiple programs simultaneously on a computer system. Without processes, a computer could only run one program at a time, making multitasking impossible. Processes provide isolation between running programs, ensuring that one program cannot interfere with another's memory or resources. They enable resource allocation, scheduling, and protection mechanisms that allow modern operating systems to safely run hundreds of programs concurrently. This abstraction is crucial for system stability, security, and efficient resource utilization, making it possible for users to run web browsers, text editors, games, and system services all at the same time.

# **Core Explanation:**

---

A process is ==an instance of a program in execution, representing the dynamic execution context of a program loaded into memory.== While a program is static code stored on disk, a process is the living, breathing execution of that program with its own memory space, system resources, and execution state.

Key characteristics of processes include:
- **Memory Space**: Each process has its own virtual address space, including code, data, heap, and stack segments
- **Process ID (PID)**: A unique identifier assigned by the operating system
- **Process Control Block (PCB)**: A data structure containing process state, program counter, CPU registers, memory management information, and I/O status
- **State Management**: Processes transition between states like new, ready, running, waiting, and terminated
- **Resource Ownership**: Each process owns resources like open files, network connections, and allocated memory

<mark style="background:

# FF5582A6;">Processes are managed by the operating system's process scheduler, which determines which process gets CPU time and when</mark>. The OS maintains process isolation through memory protection mechanisms, ensuring processes cannot access each other's memory directly. Inter-Process Communication (IPC) mechanisms like pipes, shared memory, and message queues allow controlled communication between processes when needed.

Process creation typically occurs through system calls like fork in Unix-like systems, where a parent process creates a child process. The process lifecycle involves creation, execution, and termination, with the OS handling resource allocation and cleanup throughout this lifecycle.

# **Related Concepts:**

---
**Threads**: Lightweight processes that share memory space within a single process, allowing concurrent execution with lower overhead than full processes but requiring careful synchronization.

**Process Scheduling**: The OS mechanism that determines which process runs on the CPU at any given time, using algorithms like round-robin, priority-based, or shortest job first scheduling.

**Virtual Memory**: The memory management technique that gives each process the illusion of having its own large, contiguous memory space, while the OS manages the actual physical memory allocation.

**Inter-Process Communication (IPC)**: Methods for processes to communicate and synchronize, including pipes, message queues, shared memory, and sockets, since processes are isolated by default.

**System Calls**: The interface between processes and the operating system kernel, allowing processes to request services like file I/O, process creation, and memory allocation.

**Context Switching**: The mechanism by which the OS saves the state of one process and loads the state of another, enabling multitasking by rapidly switching between processes.

**Process Synchronization**: Coordination mechanisms like semaphores, mutexes, and monitors that prevent race conditions when processes access shared resources.

# **Examples:**

---
```c
// C Example - Basic Process Creation using fork
// Demonstrates how processes are created and managed in Unix-like systems

# include <stdio.h>

# include <unistd.h>

# include <sys/wait.h>

# include <stdlib.h>

int main {
 pid_t pid;
 int status;

 printf("Parent process PID: %d\n", getpid);

 // Create a new process using fork system call
 // fork creates an exact copy of the current process
 pid = fork;

 if (pid == -1) {
 // fork failed - handle error
 perror("fork failed");
 exit(1);
 }
 else if (pid == 0) {
 // This code runs in the CHILD process
 // Child process has its own PID and memory space
 printf("Child process PID: %d\n", getpid);
 printf("Child's parent PID: %d\n", getppid);

 // Child process performs its task
 for (int i = 0; i < 5; i++) {
 printf("Child working... iteration %d\n", i);
 sleep(1); // Simulate work
 }

 printf("Child process terminating\n");
 exit(0); // Child process exits with status 0
 }
 else {
 // This code runs in the PARENT process
 // pid contains the child's process ID
 printf("Parent created child with PID: %d\n", pid);

 // Parent can continue its own work while child runs
 printf("Parent doing other work...\n");

 // Wait for child process to complete
 // This prevents zombie processes
 wait(&status);
 printf("Child process completed with status: %d\n", status);
 }

 printf("Process %d finishing\n", getpid);
 return 0;
}
````

```c
// Advanced Example - Process Communication using Pipes
// Shows how processes can communicate through IPC mechanisms

# include <stdio.h>

# include <unistd.h>

# include <string.h>

# include <sys/wait.h>

int main {
 int pipefd; // File descriptors for pipe [read_end, write_end]
 pid_t pid;
 char buffer;

 // Create a pipe before forking
 // Pipe allows data flow between parent and child processes
 if (pipe(pipefd) == -1) {
 perror("pipe failed");
 return 1;
 }

 pid = fork;

 if (pid == 0) {
 // CHILD PROCESS - acts as data producer
 close(pipefd); // Close read end in child

 char message = "Hello from child process!";

 // Child writes data to pipe
 // This data will be sent to parent process
 write(pipefd, message, strlen(message) + 1);
 printf("Child (PID %d) sent message through pipe\n", getpid);

 close(pipefd); // Close write end
 exit(0);
 }
 else {
 // PARENT PROCESS - acts as data consumer
 close(pipefd); // Close write end in parent

 // Parent reads data from pipe
 // This demonstrates inter-process communication
 read(pipefd, buffer, sizeof(buffer));
 printf("Parent (PID %d) received: %s\n", getpid, buffer);

 close(pipefd); // Close read end
 wait(NULL); // Wait for child to complete
 }

 return 0;
}
```

```python

# Python Example - Process Management and Monitoring

# Demonstrates process creation, monitoring, and resource usage

import multiprocessing
import os
import time
import psutil

# External library for system monitoring

def worker_process(process_id, shared_counter):
 """
 Worker function that runs in a separate process
 Each process has its own memory space and PID
 """
 print(f"Worker {process_id} started with PID: {os.getpid}")

# Each process performs independent work
 for i in range(5):

# Simulate CPU-intensive work
 result = sum(x*x for x in range(100000))

# Access shared resource (requires synchronization)
 with shared_counter.get_lock:
 shared_counter.value += 1
 print(f"Worker {process_id}: iteration {i+1}, counter: {shared_counter.value}")

 time.sleep(1)

 print(f"Worker {process_id} (PID: {os.getpid}) completed")

def monitor_processes(processes):
 """
 Monitor running processes and their resource usage
 """
 while any(p.is_alive for p in processes):
 print("\n
---
Process Status
---
")
 for i, process in enumerate(processes):
 if process.is_alive:

# Get process information using psutil
 try:
 p = psutil.Process(process.pid)
 cpu_percent = p.cpu_percent
 memory_info = p.memory_info

 print(f"Process {i}: PID={process.pid}, "
 f"CPU={cpu_percent}%, "
 f"Memory={memory_info.rss/1024/1024:.1f}MB")
 except psutil.NoSuchProcess:
 print(f"Process {i}: No longer exists")

 time.sleep(2)

if __name__ == "__main__":
 print(f"Main process PID: {os.getpid}")

# Create shared variable between processes

# This requires special multiprocessing types for sharing
 shared_counter = multiprocessing.Value('i', 0)

# Create multiple processes
 processes =
 for i in range(3):

# Each process runs independently with its own memory space
 p = multiprocessing.Process(
 target=worker_process,
 args=(i, shared_counter)
 )
 processes.append(p)
 p.start

# Start the process
 print(f"Started process {i} with PID: {p.pid}")

# Monitor processes in a separate thread
 import threading
 monitor_thread = threading.Thread(target=monitor_processes, args=(processes,))
 monitor_thread.start

# Wait for all processes to complete
 for i, process in enumerate(processes):
 process.join

# Wait for process to finish
 print(f"Process {i} joined (finished)")

 monitor_thread.join

 print(f"\nAll processes completed. Final counter value: {shared_counter.value}")
 print("Main process terminating")
```

# **Flashcards:**

---
What is the difference between a program and a process?;; A program is static code stored on disk, while a process is a running instance of a program in memory with its own execution context, memory space, and system resources

What are the main states a process can be in during its lifecycle?;; New (being created), Ready (waiting for CPU), Running (executing on CPU), Waiting/Blocked (waiting for I/O or event), and Terminated (finished execution)

How does the operating system maintain isolation between processes?;; Through virtual memory management that gives each process its own address space, memory protection mechanisms that prevent processes from accessing each other's memory, and the Process Control Block (PCB) that maintains separate execution contexts