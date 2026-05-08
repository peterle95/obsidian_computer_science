---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-09-18
scheda: done
visit-count: 5
confidence-level: 3
consecutive-correct: 4
last-struggle-date: 2025-08-06
cssclasses:
  - difficult
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

Non-blocking I/O solves the fundamental problem of ==server scalability and responsiveness when handling multiple concurrent connections==. In <mark style="background: #FF5582A6;">traditional blocking I/O, a server thread or process waits (blocks) until data is available to read or until a write operation completes. This approach doesn't scale well because each connection requires its own thread/process, leading to resource exhaustion and poor performance under high load.</mark>

Non-blocking I/O with proper connection management allows a single-threaded server to handle thousands of concurrent connections efficiently. Instead of waiting for individual operations to complete, the server continuously monitors all active connections and processes only those ready for I/O operations. This is essential for building high-performance web servers, real-time systems, and any application requiring concurrent network communication without the overhead of multiple threads.

# **Core Explanation:**
---

**Non-blocking I/O** means that ==I/O operations (read/write) return immediately, even if they cannot complete fully==. Instead of waiting for data to become available or for space to write data, the operation returns with either:

>- The amount of data actually processed
>- An error code indicating the operation would block (EAGAIN/EWOULDBLOCK)

**Key Components:**

**1. [[I⧸O Multiplexing]]**: Using system calls like `poll()`, [[select()]], `epoll()` (Linux), or `kqueue()` (BSD/macOS) to ==monitor multiple file descriptors simultaneously==. These calls block until at least one descriptor is ready for I/O.

**2. File Descriptor States**: Each [[Sockets]] can be in various states:

- Ready for reading (incoming data available)
- Ready for writing (can send data without blocking)
- Has an error condition
- Connection closed by peer

**3. Event-Driven Processing**: The server processes events in a loop:

- Call poll/select to wait for events
- Process all ready file descriptors
- Update connection states
- Repeat

**4. Connection State Management**: Track each connection's current state:

- Accepting new connections
- Reading HTTP request headers
- Reading request body
- Processing request
- Sending response headers
- Sending response body
- Closing connection

**Webserv Project Specific Requirements**:

- Must use exactly ONE poll() call for ALL I/O operations
- Cannot read/write without first checking with poll()
- Must handle client disconnections gracefully
- Server must never block or hang
- Must support multiple ports and concurrent connections

# **Related Concepts:**
---

**Blocking I/O**: Traditional approach where operations wait until completion. Simple but doesn't scale well with many connections.

**Threading/Multi-processing**: Alternative approach using multiple threads/processes for concurrency. More resource-intensive but sometimes simpler to implement.

**Event Loop**: The core pattern in non-blocking I/O systems where the program continuously checks for and processes events.

**[[File Descriptors]]**: Unix abstraction for I/O resources (files, sockets, pipes). In networking, each client connection gets a unique file descriptor.

**Socket Programming**: Network programming using sockets. Non-blocking I/O is commonly applied to socket operations for network servers.

**Asynchronous Programming**: Broader concept where operations don't wait for completion. Non-blocking I/O is one implementation approach.

**HTTP State Management**: Since HTTP is stateless, the server must maintain connection state separately from protocol state.

**CGI Process Management**: When executing CGI scripts, the server must manage child processes without blocking the main server loop.

# **Examples:**
---

```cpp
// Basic server structure with non-blocking I/O
#include <sys/poll.h>
#include <sys/socket.h>
#include <fcntl.h>
#include <unistd.h>
#include <vector>
#include <map>

class WebServer {
private:
    std::vector<pollfd> poll_fds;           // Array of file descriptors for poll()
    std::map<int, ClientConnection> clients; // Track each client's state
    int server_socket;                       // Listening socket
    
public:
    void run() {
        // Main event loop - the heart of non-blocking I/O
        while (true) {
            // CRITICAL: Only ONE poll() call for ALL I/O operations
            // This monitors all sockets simultaneously without blocking indefinitely
            int ready = poll(&poll_fds[0], poll_fds.size(), 1000); // 1 second timeout
            
            if (ready < 0) {
                // Handle poll error
                continue;
            }
            
            // Process all ready file descriptors
            for (size_t i = 0; i < poll_fds.size(); ++i) {
                if (poll_fds[i].revents == 0) {
                    continue; // No events on this descriptor
                }
                
                int fd = poll_fds[i].fd;
                
                if (fd == server_socket) {
                    // New connection available - won't block because poll() indicated readiness
                    handleNewConnection();
                } else {
                    // Existing client connection has activity
                    handleClientConnection(fd, poll_fds[i].revents);
                }
            }
            
            // Clean up closed connections and update poll_fds array
            cleanupConnections();
        }
    }
    
private:
    void handleNewConnection() {
        // Accept new connection - guaranteed not to block since poll() indicated readiness
        int client_fd = accept(server_socket, NULL, NULL);
        if (client_fd < 0) {
            return; // Handle error
        }
        
        // CRITICAL: Set client socket to non-blocking mode
        int flags = fcntl(client_fd, F_GETFL, 0);
        fcntl(client_fd, F_SETFL, flags | O_NONBLOCK);
        
        // Add to poll array for monitoring
        pollfd pfd = {client_fd, POLLIN, 0}; // Monitor for incoming data
        poll_fds.push_back(pfd);
        
        // Initialize client state
        clients[client_fd] = ClientConnection(STATE_READING_HEADERS);
    }
    
    void handleClientConnection(int fd, short revents) {
        ClientConnection& client = clients[fd];
        
        if (revents & POLLIN) {
            // Data available for reading - process based on current state
            handleClientRead(fd, client);
        }
        
        if (revents & POLLOUT) {
            // Socket ready for writing - send response data
            handleClientWrite(fd, client);
        }
        
        if (revents & (POLLHUP | POLLERR)) {
            // Connection closed or error - clean up
            closeConnection(fd);
        }
    }
};
```

```cpp
// Example of proper non-blocking read operation
void handleClientRead(int fd, ClientConnection& client) {
    char buffer[4096];
    
    // Attempt to read - this won't block because poll() indicated data is ready
    ssize_t bytes_read = read(fd, buffer, sizeof(buffer));
    
    if (bytes_read > 0) {
        // Successfully read data
        client.appendData(buffer, bytes_read);
        
        // Process data based on connection state
        switch (client.state) {
            case STATE_READING_HEADERS:
                if (client.hasCompleteHeaders()) {
                    client.state = STATE_READING_BODY;
                    // Update poll events to continue monitoring
                    updatePollEvents(fd, POLLIN);
                }
                break;
                
            case STATE_READING_BODY:
                if (client.hasCompleteRequest()) {
                    client.state = STATE_PROCESSING;
                    processRequest(client);
                    client.state = STATE_WRITING_RESPONSE;
                    // Now monitor for write readiness
                    updatePollEvents(fd, POLLOUT);
                }
                break;
        }
    } else if (bytes_read == 0) {
        // Client closed connection gracefully
        closeConnection(fd);
    } else {
        // bytes_read < 0 - check errno
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            // No more data available right now - this is normal
            // Continue monitoring this connection
            return;
        } else {
            // Real error occurred
            closeConnection(fd);
        }
    }
}
```

```cpp
// Example of proper non-blocking write operation
void handleClientWrite(int fd, ClientConnection& client) {
    if (client.response_buffer.empty()) {
        return; // Nothing to send
    }
    
    // Attempt to write - won't block because poll() indicated socket is ready
    ssize_t bytes_written = write(fd, 
                                 client.response_buffer.data(), 
                                 client.response_buffer.size());
    
    if (bytes_written > 0) {
        // Successfully wrote some data
        client.response_buffer.erase(0, bytes_written);
        
        if (client.response_buffer.empty()) {
            // Response completely sent
            if (client.should_keep_alive) {
                // HTTP/1.1 keep-alive - reset for next request
                client.reset();
                client.state = STATE_READING_HEADERS;
                updatePollEvents(fd, POLLIN); // Monitor for next request
            } else {
                // Close connection after response
                closeConnection(fd);
            }
        }
        // If buffer not empty, continue monitoring for POLLOUT
    } else if (bytes_written < 0) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            // Socket not ready for more data - continue monitoring
            return;
        } else {
            // Real error occurred
            closeConnection(fd);
        }
    }
}
```

```cpp
// Connection state management example
enum ConnectionState {
    STATE_READING_HEADERS,
    STATE_READING_BODY,
    STATE_PROCESSING,
    STATE_WRITING_RESPONSE,
    STATE_CLOSING
};

class ClientConnection {
public:
    ConnectionState state;
    std::string request_buffer;    // Accumulated request data
    std::string response_buffer;   // Response data to send
    bool should_keep_alive;        // HTTP/1.1 connection management
    time_t last_activity;          // For timeout handling
    
    ClientConnection(ConnectionState initial_state = STATE_READING_HEADERS) 
        : state(initial_state), should_keep_alive(false) {
        last_activity = time(NULL);
    }
    
    void appendData(const char* data, size_t len) {
        request_buffer.append(data, len);
        last_activity = time(NULL); // Update activity timestamp
    }
    
    bool hasCompleteHeaders() {
        // Look for double CRLF indicating end of headers
        return request_buffer.find("\r\n\r\n") != std::string::npos;
    }
    
    bool hasCompleteRequest() {
        // Check if we have complete request based on Content-Length
        // or chunked encoding completion
        return true; // Simplified for example
    }
    
    void reset() {
        // Reset for next request on persistent connection
        request_buffer.clear();
        response_buffer.clear();
        state = STATE_READING_HEADERS;
    }
};
```

# **Flashcards:**
---

What is the key difference between blocking and non-blocking I/O?;; Blocking I/O waits (blocks) until an operation can complete fully, potentially freezing the program. Non-blocking I/O returns immediately, either with data/success or an indication that the operation would block (EAGAIN/EWOULDBLOCK), allowing the program to continue processing other tasks.

What is the role of poll(), select(), or epoll() in non-blocking I/O?;; These system calls monitor multiple file descriptors simultaneously and block until at least one is ready for I/O operations. They enable a single thread to handle many connections efficiently by only processing descriptors that are actually ready for reading, writing, or have errors.

Why does the webserv project require exactly ONE poll() call for all I/O operations?;; Using a single poll() call ensures true non-blocking behavior and efficient resource usage. Multiple poll() calls would fragment the monitoring process and could lead to blocking behavior. The single poll() monitors all sockets (server listening socket + all client connections) simultaneously.

What are the main connection states a web server must track for each client?;; Key states include: accepting new connections, reading HTTP request headers, reading request body, processing the request, sending response headers, sending response body, and closing the connection. Each state determines what I/O operations are needed and what poll() events to monitor.

What does EAGAIN/EWOULDBLOCK mean in non-blocking I/O and how should it be handled?;; EAGAIN/EWOULDBLOCK indicates that a read() or write() operation cannot complete immediately without blocking. This is normal in non-blocking I/O. The correct response is to continue monitoring the file descriptor with poll() and try the operation again when poll() indicates readiness.

How does connection management differ between HTTP/1.0 and HTTP/1.1 in a non-blocking server?;; HTTP/1.0 typically closes connections after each response, requiring simpler state management. HTTP/1.1 supports persistent connections (keep-alive), so the server must reset the connection state after sending a response and continue monitoring for the next request on the same socket rather than closing it.