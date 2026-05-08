---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-10-24
scheda: done
visit-count: 7
confidence-level: 4
consecutive-correct: 6
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

I/O Multiplexing solves the fundamental problem of ==efficiently handling multiple I/O streams simultaneously in a single thread or process.== Without I/O multiplexing, applications face the "C10K problem" - how to handle thousands of concurrent connections efficiently.

Traditional approaches have critical limitations:

1. **Sequential Processing**: Handle one connection at a time - extremely slow and unscalable
2. **Multi-threading**: Create one thread per connection - resource exhaustive, context switching overhead, synchronization complexity
3. **Multi-processing**: Create one process per connection - even more resource intensive than threading

<mark style="background: #FF5582A6;">I/O Multiplexing enables a single thread to monitor hundreds or thousands of file descriptors (sockets, files, pipes) simultaneously, processing only those that are ready for I/O operations. </mark>This is essential for building high-performance servers, real-time systems, and any application requiring concurrent I/O without the overhead of multiple threads or processes. It's the foundation of modern event-driven architectures used by web servers like NGINX, databases like Redis, and networking libraries.

# **Core Explanation:**
---

**I/O Multiplexing** is a technique that allows<mark style="background: #ADCCFFA6;"> a single thread to monitor multiple file descriptors simultaneously and determine which ones are ready for I/O operations without blocking.</mark> <mark style="background: #ADCCFFA6;">Instead of checking each file descriptor individually (which would block if not ready), multiplexing uses system calls that can monitor many descriptors at once.</mark>

**Core Concept**: "<mark style="background: #FF5582A6;">Wait for any of N file descriptors to become ready, then process only the ready ones.</mark>"

**Key Characteristics:**

**1. Event-Driven Architecture**: The application reacts to I/O events as they occur, rather than proactively polling or blocking on individual operations.

**2. Single-Threaded Efficiency**:<mark style="background: #FFB86CA6;"> One thread can handle thousands of connections by only processing those that are actually ready for I/O.</mark>

**3. Non-Blocking I/O Integration**: Typically used with non-blocking file descriptors to prevent individual operations from blocking after readiness is confirmed.

**4. Scalability**: Scales much better than thread-per-connection models, with O(1) or O(log n) complexity depending on the mechanism used.

**Main I/O Multiplexing Mechanisms:**

**[[select()]]**: Original UNIX mechanism using file descriptor sets (fd_set). Limited to FD_SETSIZE descriptors (~1024), uses bit manipulation.

**poll()**: Improved interface using an array of pollfd structures. No descriptor limit, cleaner API than select().

**epoll() (Linux)**: Most efficient for large numbers of descriptors. Uses event notification instead of polling all descriptors.

**kqueue() (BSD/macOS)**: BSD equivalent to epoll(), highly efficient event notification system.

**How It Works:**

>1. Application ==maintains a collection of file descriptors to monitor==
>2. <mark style="background: #BBFABBA6;">Calls a multiplexing system call</mark> (poll, select, epoll, kqueue) specifying which events to monitor
>3. <mark style="background: #FF5582A6;">System call blocks until at least one descriptor is ready or timeout occurs</mark>
>4. Application <mark style="background: #D2B3FFA6;">processes all ready descriptors</mark>
>5. <mark style="background: #ADCCFFA6;">Updates the monitoring set as connections are added/removed</mark>
>6. Repeats the cycle

**Event Types Monitored:**

- **Read Readiness**: Data available to read without blocking
- **Write Readiness**: Space available to write without blocking
- **Exception/Error**: Error conditions or out-of-band data
- **Hangup**: Connection closed by peer

## **Webserv Project Context**

For the webserv project, I/O multiplexing is mandatory and has specific requirements:

- Must use poll() or equivalent (select(), epoll(), kqueue()) for ALL I/O operations
- Single multiplexing call must handle server listening socket + all client connections
- Must monitor both reading and writing events simultaneously
- Cannot perform any read/write without first checking readiness through multiplexing
- Must handle client disconnections and errors gracefully
- Server must support multiple ports and concurrent connections efficiently

# **Related Concepts:**
---

**Non-Blocking I/O**: Complementary technique where individual I/O operations return immediately rather than blocking. I/O multiplexing determines WHEN to perform operations, non-blocking I/O ensures individual operations don't block.

**Event Loop**: The programming pattern that implements I/O multiplexing - continuously check for events and process them. The event loop is the architectural pattern, I/O multiplexing is the underlying technique.

**Asynchronous I/O (AIO)**: Alternative approach where the kernel performs I/O operations and notifies completion. I/O multiplexing is synchronous (you wait for readiness), AIO is asynchronous (kernel does work for you).

**Reactor Pattern**: Design pattern that demultiplexes and dispatches service requests. I/O multiplexing is the mechanism, Reactor is the architectural pattern that uses it.

**Thread Pools vs I/O Multiplexing**: Alternative concurrency approaches. Thread pools use multiple threads with blocking I/O, multiplexing uses single thread with non-blocking I/O. Each has trade-offs in complexity, resource usage, and performance characteristics.

**File Descriptors**: The abstraction that I/O multiplexing operates on. Every socket, file, or pipe gets a file descriptor that can be monitored.

**Socket Programming**: Network programming context where I/O multiplexing is most commonly applied. Servers use multiplexing to handle many client connections simultaneously.

**Signal-Driven I/O**: Alternative notification mechanism using SIGIO signals. Less portable and harder to use than I/O multiplexing.

# **Examples:**
---

```cpp
// Conceptual comparison: Traditional blocking vs I/O Multiplexing approach

// TRADITIONAL BLOCKING APPROACH (doesn't scale):
void traditionalServer() {
    int server_fd = createServerSocket();
    
    while (true) {
        // Accept blocks until new connection arrives
        int client_fd = accept(server_fd, NULL, NULL);
        
        // Handle one client at a time - other clients must wait!
        char buffer[1024];
        ssize_t n = read(client_fd, buffer, sizeof(buffer)); // Blocks until data arrives
        
        // Process request...
        processRequest(buffer, n);
        
        // Send response
        write(client_fd, response.c_str(), response.length()); // May block if client slow
        
        close(client_fd);
        // Next client can only be handled after this one is complete!
    }
}

// I/O MULTIPLEXING APPROACH (scales to thousands):
void multiplexingServer() {
    int server_fd = createServerSocket();
    fd_set read_fds, write_fds, master_fds;
    int max_fd = server_fd;
    
    FD_ZERO(&master_fds);
    FD_SET(server_fd, &master_fds);
    
    while (true) {
        // Copy master set for select() call
        read_fds = master_fds;
        FD_ZERO(&write_fds);
        // Set write_fds for clients that need to send data
        
        // SINGLE call monitors ALL file descriptors simultaneously
        // This is the essence of I/O multiplexing
        int ready = select(max_fd + 1, &read_fds, &write_fds, NULL, NULL);
        
        // Process ALL ready descriptors in one cycle
        for (int fd = 0; fd <= max_fd; ++fd) {
            if (FD_ISSET(fd, &read_fds)) {
                if (fd == server_fd) {
                    // New connection ready - won't block
                    int client_fd = handleNewConnection();
                    FD_SET(client_fd, &master_fds);
                    if (client_fd > max_fd) max_fd = client_fd;
                } else {
                    // Client data ready - won't block
                    if (!handleClientData(fd)) {
                        // Client disconnected
                        FD_CLR(fd, &master_fds);
                        close(fd);
                    }
                }
            }
            if (FD_ISSET(fd, &write_fds)) {
                // Client ready for writing - won't block
                handleClientWrite(fd);
            }
        }
        // Can handle HUNDREDS of clients in single iteration!
    }
}
```

```cpp
// Comprehensive I/O Multiplexing example for webserv architecture
#include <poll.h>
#include <vector>
#include <map>
#include <iostream>

class IOMultiplexingServer {
private:
    std::vector<pollfd> monitored_fds;     // Central multiplexing array
    std::map<int, ClientState> clients;    // Track client states
    std::vector<int> server_sockets;       // Multiple listening sockets
    
public:
    void addServerSocket(int port) {
        int server_fd = createServerSocket(port);
        server_sockets.push_back(server_fd);
        
        // Add to multiplexing set - monitor for new connections
        pollfd pfd = {server_fd, POLLIN, 0};
        monitored_fds.push_back(pfd);
        
        std::cout << "Added server socket " << server_fd 
                  << " for port " << port << " to multiplexing set" << std::endl;
    }
    
    void runEventLoop() {
        std::cout << "Starting I/O multiplexing event loop..." << std::endl;
        
        while (true) {
            // THE CORE OF I/O MULTIPLEXING:
            // Single system call monitors ALL file descriptors
            // - All server sockets (multiple ports)
            // - All client connections
            // - Both reading and writing events
            int ready_count = poll(&monitored_fds[0], monitored_fds.size(), 1000);
            
            if (ready_count < 0) {
                perror("poll() failed");
                break;
            } else if (ready_count == 0) {
                // Timeout - good time for maintenance tasks
                handleTimeout();
                continue;
            }
            
            std::cout << "I/O Multiplexing detected " << ready_count 
                      << " ready file descriptors" << std::endl;
            
            // Process all ready file descriptors
            // This is where multiplexing shines - we only process what's ready
            processReadyDescriptors();
        }
    }
    
private:
    void processReadyDescriptors() {
        // Iterate through monitored descriptors to find ready ones
        for (size_t i = 0; i < monitored_fds.size(); ++i) {
            pollfd& pfd = monitored_fds[i];
            
            if (pfd.revents == 0) {
                continue; // No events on this descriptor
            }
            
            std::cout << "Processing events on fd " << pfd.fd 
                      << " (events: " << pfd.revents << ")" << std::endl;
            
            // Determine what type of descriptor this is
            if (isServerSocket(pfd.fd)) {
                handleServerSocketEvents(pfd);
            } else {
                handleClientSocketEvents(pfd, i);
            }
            
            // Clear events for next poll() call
            pfd.revents = 0;
        }
    }
    
    void handleServerSocketEvents(pollfd& pfd) {
        if (pfd.revents & POLLIN) {
            // New connection available - I/O multiplexing told us it's ready!
            acceptNewConnections(pfd.fd);
        }
        
        if (pfd.revents & (POLLERR | POLLNVAL)) {
            std::cerr << "Error on server socket " << pfd.fd << std::endl;
        }
    }
    
    void acceptNewConnections(int server_fd) {
        // Accept all available connections (there might be multiple)
        while (true) {
            int client_fd = accept(server_fd, NULL, NULL);
            
            if (client_fd < 0) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    // No more connections available
                    break;
                } else {
                    perror("accept() error");
                    break;
                }
            }
            
            // Configure new client for non-blocking I/O
            makeNonBlocking(client_fd);
            
            // ADD TO MULTIPLEXING SET - this is key!
            // Now this client will be monitored in future poll() calls
            pollfd client_pfd = {client_fd, POLLIN, 0};
            monitored_fds.push_back(client_pfd);
            
            // Initialize client state
            clients[client_fd] = ClientState();
            
            std::cout << "Added new client " << client_fd 
                      << " to I/O multiplexing set" << std::endl;
        }
    }
    
    void handleClientSocketEvents(pollfd& pfd, size_t index) {
        int client_fd = pfd.fd;
        
        // Handle error conditions first
        if (pfd.revents & (POLLERR | POLLHUP | POLLNVAL)) {
            std::cout << "Client " << client_fd << " disconnected or error" << std::endl;
            removeFromMultiplexingSet(client_fd, index);
            return;
        }
        
        // Handle read events
        if (pfd.revents & POLLIN) {
            if (!handleClientRead(client_fd)) {
                // Client disconnected during read
                removeFromMultiplexingSet(client_fd, index);
                return;
            }
            
            // Check if we now have a response ready to send
            if (clients[client_fd].hasResponseReady()) {
                // Modify multiplexing to also monitor for write readiness
                pfd.events |= POLLOUT;
            }
        }
        
        // Handle write events
        if (pfd.revents & POLLOUT) {
            if (handleClientWrite(client_fd)) {
                // Response fully sent, stop monitoring for writes
                pfd.events &= ~POLLOUT;
                
                // For HTTP/1.1 keep-alive, continue monitoring for reads
                // For HTTP/1.0 or Connection: close, remove from set
                if (!clients[client_fd].keepAlive()) {
                    removeFromMultiplexingSet(client_fd, index);
                }
            }
        }
    }
    
    void removeFromMultiplexingSet(int fd, size_t index) {
        // REMOVE FROM MULTIPLEXING SET - crucial for cleanup
        close(fd);
        clients.erase(fd);
        
        // Efficient removal: swap with last element and pop
        if (index < monitored_fds.size() - 1) {
            monitored_fds[index] = monitored_fds.back();
        }
        monitored_fds.pop_back();
        
        std::cout << "Removed fd " << fd << " from I/O multiplexing set" << std::endl;
    }
};
```

```cpp
// Advanced example: Dynamic event management in I/O multiplexing
class DynamicMultiplexing {
private:
    std::vector<pollfd> fds;
    std::map<int, ConnectionState> states;
    
public:
    // Core multiplexing function with dynamic event management
    void processConnections() {
        while (true) {
            // Prepare events based on current connection states
            updateMultiplexingEvents();
            
            // The multiplexing call - monitors all at once
            int ready = poll(&fds[0], fds.size(), 500);
            
            if (ready > 0) {
                handleMultiplexingResults();
            }
            
            // Clean up finished connections
            cleanupConnections();
        }
    }
    
private:
    void updateMultiplexingEvents() {
        // Dynamically adjust what events we monitor based on connection state
        for (size_t i = 0; i < fds.size(); ++i) {
            int fd = fds[i].fd;
            ConnectionState& state = states[fd];
            
            // Start with no events
            fds[i].events = 0;
            
            // Add read monitoring based on state
            if (state.needsMoreInput()) {
                fds[i].events |= POLLIN;
                std::cout << "Monitoring fd " << fd << " for reading" << std::endl;
            }
            
            // Add write monitoring based on state
            if (state.hasDataToSend()) {
                fds[i].events |= POLLOUT;
                std::cout << "Monitoring fd " << fd << " for writing" << std::endl;
            }
            
            // This dynamic adjustment is what makes I/O multiplexing efficient:
            // We only monitor for events that are actually meaningful
            // for each connection's current state
        }
    }
    
    void handleMultiplexingResults() {
        for (size_t i = 0; i < fds.size(); ++i) {
            if (fds[i].revents == 0) continue;
            
            int fd = fds[i].fd;
            short events = fds[i].revents;
            
            std::cout << "I/O multiplexing found activity on fd " << fd << std::endl;
            
            // Process events in priority order
            if (events & (POLLERR | POLLHUP)) {
                handleConnectionError(fd);
            } else {
                if (events & POLLIN) {
                    handleReadEvent(fd);
                }
                if (events & POLLOUT) {
                    handleWriteEvent(fd);
                }
            }
            
            fds[i].revents = 0; // Clear for next iteration
        }
    }
    
    // Example of state-driven multiplexing decisions
    void handleReadEvent(int fd) {
        ConnectionState& state = states[fd];
        
        // Read available data
        char buffer[4096];
        ssize_t n = read(fd, buffer, sizeof(buffer));
        
        if (n > 0) {
            state.appendData(buffer, n);
            
            // State transition affects future multiplexing
            if (state.hasCompleteRequest()) {
                // No longer need to monitor for reads
                // Will monitor for writes when response is ready
                state.processRequest();
                std::cout << "Request complete on fd " << fd 
                          << " - will monitor for write readiness" << std::endl;
            }
        } else if (n == 0) {
            // Client closed - will be removed from multiplexing set
            state.markClosed();
        }
    }
    
    void handleWriteEvent(int fd) {
        ConnectionState& state = states[fd];
        
        // Send available response data
        ssize_t n = write(fd, state.getResponseData(), state.getResponseSize());
        
        if (n > 0) {
            state.advanceResponse(n);
            
            if (state.responseComplete()) {
                std::cout << "Response sent on fd " << fd << std::endl;
                
                // HTTP connection management affects multiplexing
                if (state.shouldKeepAlive()) {
                    // Reset for next request - monitor for reads again
                    state.resetForNextRequest();
                } else {
                    // Mark for removal from multiplexing set
                    state.markForClosing();
                }
            }
        }
    }
};
```

```cpp
// Comparison of different I/O multiplexing mechanisms
void demonstrateMultiplexingOptions() {
    std::vector<int> client_fds = {3, 4, 5, 6, 7}; // Example client sockets
    
    // 1. Using select() - original multiplexing mechanism
    {
        fd_set read_fds, write_fds;
        int max_fd = 0;
        
        // Prepare file descriptor sets
        FD_ZERO(&read_fds);
        FD_ZERO(&write_fds);
        
        for (int fd : client_fds) {
            FD_SET(fd, &read_fds);    // Monitor for reading
            if (hasDataToSend(fd)) {
                FD_SET(fd, &write_fds); // Monitor for writing
            }
            max_fd = std::max(max_fd, fd);
        }
        
        // Multiplexing call - monitors all at once
        int ready = select(max_fd + 1, &read_fds, &write_fds, NULL, NULL);
        
        // Process results
        if (ready > 0) {
            for (int fd : client_fds) {
                if (FD_ISSET(fd, &read_fds)) {
                    handleRead(fd);
                }
                if (FD_ISSET(fd, &write_fds)) {
                    handleWrite(fd);
                }
            }
        }
    }
    
    // 2. Using poll() - cleaner interface (recommended for webserv)
    {
        std::vector<pollfd> fds;
        
        // Prepare poll array
        for (int fd : client_fds) {
            pollfd pfd = {fd, POLLIN, 0};
            if (hasDataToSend(fd)) {
                pfd.events |= POLLOUT;
            }
            fds.push_back(pfd);
        }
        
        // Multiplexing call - monitors all at once
        int ready = poll(&fds[0], fds.size(), -1);
        
        // Process results
        if (ready > 0) {
            for (size_t i = 0; i < fds.size(); ++i) {
                if (fds[i].revents & POLLIN) {
                    handleRead(fds[i].fd);
                }
                if (fds[i].revents & POLLOUT) {
                    handleWrite(fds[i].fd);
                }
            }
        }
    }
    
    // The key insight: All mechanisms achieve the same goal:
    // Monitor multiple file descriptors simultaneously,
    // process only those that are ready,
    // scale to handle many connections efficiently
}
```

# **Flashcards:**
---

What is I/O Multiplexing and what fundamental problem does it solve?;; I/O Multiplexing is a technique that allows a single thread to monitor multiple file descriptors simultaneously and process only those ready for I/O operations. It solves the C10K problem by enabling efficient handling of thousands of concurrent connections without the resource overhead of multiple threads or processes.

What are the main I/O multiplexing mechanisms available and their key differences?;; select() (original, limited to ~1024 descriptors, uses bit manipulation), poll() (cleaner array interface, no descriptor limit), epoll() (Linux-specific, most efficient for large numbers), and kqueue() (BSD/macOS, similar efficiency to epoll). Each trades complexity for performance and portability.

How does I/O Multiplexing differ from traditional multi-threading approaches?;; I/O Multiplexing uses a single thread with non-blocking I/O to handle many connections, while multi-threading uses multiple threads with blocking I/O. Multiplexing has lower memory overhead, no synchronization issues, and better cache locality, but requires more complex state management.

What is the relationship between I/O Multiplexing and the Event Loop pattern?;; The Event Loop is an architectural pattern that implements I/O Multiplexing. The event loop continuously: 1) calls a multiplexing function to wait for events, 2) processes all ready file descriptors, 3) updates the monitoring set, 4) repeats. I/O Multiplexing is the mechanism, Event Loop is the pattern.

What types of events can I/O Multiplexing monitor and why is this important?;; I/O Multiplexing can monitor read readiness (data available), write readiness (space available to send), error conditions, and connection hangups. This allows applications to respond precisely to the type of I/O operation that's possible, avoiding blocking operations and enabling efficient resource utilization.

How does I/O Multiplexing enable the webserv project requirements?;; I/O Multiplexing allows webserv to use a single poll() call to monitor all sockets (server listening sockets + client connections) for both reading and writing simultaneously. This enables handling multiple ports and concurrent connections efficiently while ensuring no operation blocks, meeting the project's non-blocking I/O requirements.