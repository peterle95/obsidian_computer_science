---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-10-02
scheda: done
visit-count: 5
confidence-level: 3
consecutive-correct: 4
last-struggle-date: 2025-08-11
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

File descriptors solve the fundamental problem of **unified I/O abstraction** in Unix-like systems. They <mark style="background: #FF5582A6;">provide a single, consistent interface for interacting with different types of resources </mark>(files, network [[Sockets]], pipes, devices) <mark style="background: #FF5582A6;">through simple integer identifiers</mark>. <mark style="background: #BBFABBA6;">This abstraction allows programs to use the same system calls (read, write, close) regardless of whether they're working with a regular file, a network connection, or standard input/output</mark>. File descriptors are crucial for system programming, networking, and building scalable applications that need to handle multiple I/O operations efficiently.

# **Core Explanation:**
---

A **file descriptor** is a non-negative integer that serves as a unique identifier for an open file or I/O resource within a process. ==The operating system maintains a **file descriptor table** for each process, where each entry points to a **file table entry** that contains metadata about the open resource.==

**Key characteristics:**

- **Process-specific**: <mark style="background: #ABF7F7A6;">Each process has its own file descriptor table</mark>
- **Integer identifiers**: Typically start from 0 and increment
- **Standard descriptors**: 0 (stdin), 1 (stdout), 2 (stderr) are pre-opened
- **Resource abstraction**: Can represent files, sockets, pipes, devices
- **Kernel-managed**: The OS allocates and tracks file descriptors
- **Limited resource**: Each process has a maximum number of file descriptors

**How it works:**

>1. **Creation:** A program requests the operating system to open a file, create a pipe, or establish a network connection (e.g., using `open()`, `pipe()`, `socket()` system calls).
>2. **Assignment:** If the request is successful, the kernel returns a file descriptor (an integer) to the program. This integer is an index into a table maintained by the kernel for that process, which stores information about the open resource.
>3. **Operation:** The program then uses this file descriptor in subsequent system calls (like `read()`, `write()`, `close()`) to perform operations on the associated resource. For example, data can be read from a file or received from a network connection, or data can be written to a file or sent over a network connection.
>4. **Closure:** When the program is finished with the resource, it calls `close()` with the file descriptor, releasing the resource and freeing up the descriptor.
## **Webserv Project Relevance:**
---

For the `webserv` project, the concept of file descriptors is **central to its core requirements and implementation**. The project explicitly mandates building a non-blocking HTTP server in C++98 that can handle multiple clients concurrently.

• **Single Multiplexing Call:** The server **"must use only 1 poll() (or equivalent) for all the I/O operations between the clients and the server (listen included)"**. This means a single `pollfd` array (for `poll()`) or `fd_set` (for [[select()]]) must be used to monitor all active client connections and the listening socket.

• **Monitoring Read and Write:** The `poll()` (or equivalent) call **"must monitor both reading and writing simultaneously"**. This means that for each client socket, the server must determine if it's ready to receive incoming requests (`POLLIN`) or ready to send responses (`POLLOUT`).

• **Strict I/O Handling:** It is forbidden to "do a read or a write operation without going through poll() (or equivalent)". This reinforces the importance of using the multiplexing mechanism to determine readiness before attempting I/O, preventing blocking.

• **Handling Multiple Clients:** The server needs an "array of socket descriptors" to manage numerous users accessing the website simultaneously. When a new connection arrives, `accept()` creates a new socket file descriptor, which then needs to be added to the set of file descriptors being monitored by `poll()` or `select()`.

• **Specific System Calls:** The project's allowed external functions list many system calls that directly involve file descriptors, such as `socket()`, `accept()`, `listen()`, `send()`, `recv()`, `bind()`, `setsockopt()`, `close()`, `read()`, and `write()`. Understanding how these functions operate on file descriptors is fundamental to implementing the server.

**Server Socket Management:**

```cpp
// Server listens on multiple ports using file descriptors
int server_fd = socket(AF_INET, SOCK_STREAM, 0); // Create listening socket
bind(server_fd, (struct sockaddr*)&address, sizeof(address)); // Bind to port
listen(server_fd, SOMAXCONN); // Start listening for connections
```

**Client Connection Handling:**

```cpp
// Each client connection gets its own file descriptor
int client_fd = accept(server_fd, (struct sockaddr*)&client_addr, &addr_len);
// client_fd is used to read requests and send responses to that specific client
```

**Non-blocking I/O Requirements:** The project mandates non-blocking file descriptors and using poll() (or equivalent) for all I/O. The `webserv` project requires the server to be **"non-blocking at all times"** and to handle client disconnections properly. This is achieved by setting socket file descriptors to non-blocking mode and using I/O multiplexing functions.

```cpp
// Set file descriptors to non-blocking mode
fcntl(client_fd, F_SETFL, O_NONBLOCK);

// Use poll() to monitor multiple file descriptors
struct pollfd fds[MAX_CLIENTS];
fds[i].fd = client_fd;
fds[i].events = POLLIN | POLLOUT; // Monitor for read/write readiness
poll(fds, num_fds, timeout);
```

**File Serving:**

```cpp
// Serve static files using file descriptors
int file_fd = open("index.html", O_RDONLY);
// Read file content and send to client_fd
```

**CGI Execution:**

Even for optional features like Common Gateway Interface (CGI), file descriptors play a role, as the server needs to provide the full client request and arguments to the CGI via environment variables and then read the CGI's output. This likely involves piping data, which also uses file descriptors.

```cpp
// CGI scripts require pipe file descriptors for communication
int pipe_fds[2];
pipe(pipe_fds); // Create pipe for CGI input/output
// Child process uses pipe_fds to communicate with CGI script
```

**Key webserv considerations:**

- Must handle multiple client connections simultaneously (multiple file descriptors)
- Cannot block on any I/O operation - must use poll()/select()/epoll()
- Need to track and manage many file descriptors efficiently
- Must handle client disconnections gracefully (closing file descriptors)
- File descriptor limits affect maximum concurrent connections

In essence, the `webserv` project challenges developers to deeply understand and practically apply the concept of file descriptors and associated I/O multiplexing techniques to build a robust, high-performance web server from scratch.
# **Related Concepts:**

---

- **File Table**: Kernel structure containing metadata about open files; file descriptors index into this table
- **Inode**: Filesystem structure containing file metadata; file table entries point to inodes
- **Process Control Block (PCB)**: Contains the file descriptor table for each process
- **System Calls**: Functions like open(), read(), write(), close() that operate on file descriptors
- **Socket Programming**: Network sockets are accessed through file descriptors
- **Pipes and IPC**: Inter-process communication mechanisms that use file descriptors
- **I/O Multiplexing**: Techniques like select(), poll(), epoll() that monitor multiple file descriptors
- **Non-blocking I/O**: File descriptors can be configured for non-blocking operations
- **File Permissions**: Access rights that determine what operations are allowed on a file descriptor

# **Examples:**
---

```cpp
#include <fcntl.h>
#include <unistd.h>
#include <iostream>
#include <sys/socket.h>

int main() {
    // Standard file descriptors (automatically opened for every process)
    // 0 = stdin, 1 = stdout, 2 = stderr
    write(1, "Hello stdout\n", 13); // Write to standard output using fd 1
    
    // Opening a regular file - returns lowest available file descriptor
    int file_fd = open("example.txt", O_CREAT | O_WRONLY, 0644);
    if (file_fd == -1) {
        perror("Failed to open file");
        return 1;
    }
    // file_fd will likely be 3 (first available after 0,1,2)
    
    // Write data to the file using the file descriptor
    const char* data = "File descriptor example\n";
    ssize_t bytes_written = write(file_fd, data, strlen(data));
    
    // Create a socket - also returns a file descriptor
    int socket_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (socket_fd == -1) {
        perror("Failed to create socket");
        close(file_fd);
        return 1;
    }
    // socket_fd will likely be 4 (next available)
    
    // Both file and socket can be used with same system calls
    // because they're both represented as file descriptors
    
    // Create a pipe - returns two file descriptors
    int pipe_fds[2];
    if (pipe(pipe_fds) == -1) {
        perror("Failed to create pipe");
        close(file_fd);
        close(socket_fd);
        return 1;
    }
    // pipe_fds[0] is read end, pipe_fds[1] is write end
    
    // Demonstrate file descriptor duplication
    int dup_fd = dup(file_fd); // Creates duplicate of file_fd
    // Both file_fd and dup_fd refer to the same file
    
    // Set file descriptor to non-blocking mode
    int flags = fcntl(socket_fd, F_GETFL, 0);
    fcntl(socket_fd, F_SETFL, flags | O_NONBLOCK);
    
    // Clean up - close all file descriptors
    close(file_fd);     // Close original file
    close(dup_fd);      // Close duplicate
    close(socket_fd);   // Close socket
    close(pipe_fds[0]); // Close pipe read end
    close(pipe_fds[1]); // Close pipe write end
    
    return 0;
}
```

```cpp
// Example of I/O multiplexing with select() - monitoring multiple file descriptors
#include <sys/select.h>
#include <unistd.h>
#include <iostream>

void io_multiplexing_example() {
    int socket1 = socket(AF_INET, SOCK_STREAM, 0);
    int socket2 = socket(AF_INET, SOCK_STREAM, 0);
    
    // Set up file descriptor set for select()
    fd_set read_fds;
    FD_ZERO(&read_fds);           // Clear the set
    FD_SET(socket1, &read_fds);   // Add socket1 to the set
    FD_SET(socket2, &read_fds);   // Add socket2 to the set
    FD_SET(0, &read_fds);         // Add stdin (fd 0) to the set
    
    // Find the highest file descriptor number
    int max_fd = std::max({socket1, socket2, 0}) + 1;
    
    // Monitor multiple file descriptors simultaneously
    int activity = select(max_fd, &read_fds, NULL, NULL, NULL);
    
    if (activity > 0) {
        // Check which file descriptors are ready for reading
        if (FD_ISSET(0, &read_fds)) {
            // stdin has data available
            char buffer[1024];
            read(0, buffer, sizeof(buffer));
        }
        if (FD_ISSET(socket1, &read_fds)) {
            // socket1 has data available
            // Handle socket1 data...
        }
        if (FD_ISSET(socket2, &read_fds)) {
            // socket2 has data available
            // Handle socket2 data...
        }
    }
    
    close(socket1);
    close(socket2);
}
```

# **Flashcards:**
---

What is a file descriptor?;; A non-negative integer that serves as a unique identifier for an open file or I/O resource within a process, providing a unified interface for different types of resources.

What are the three standard file descriptors automatically opened for every process?;; 0 (stdin - standard input), 1 (stdout - standard output), and 2 (stderr - standard error).

How does the kernel assign file descriptor numbers?;; The kernel assigns the lowest available non-negative integer as the file descriptor when a resource is opened.

What system calls commonly work with file descriptors?;; open(), close(), read(), write(), dup(), dup2(), fcntl(), select(), poll(), and epoll().

Why are file descriptors process-specific?;; Each process has its own file descriptor table maintained by the OS, so the same file descriptor number in different processes refers to different resources.

In the webserv project, why must file descriptors be non-blocking?;; To prevent the server from hanging on I/O operations and to handle multiple clients simultaneously without blocking on any single client's read/write operations.