---
memory: to_finish
tags:
  - to_learn
language:
  - Core Concepts
review-date: 2025-11-20
last-reviewed: 2025-10-21
scheda: done
visit-count: 6
confidence-level: 2.5
consecutive-correct: 3
last-struggle-date: 2025-09-03
cssclasses:
  - important
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

**Sockets** are a fundamental mechanism that most popular <mark style="background: #D2B3FFA6;">operating systems provide to give programs access to the network.</mark> They serve as the ==**communication link between two processes on a network**==, <mark style="background: #D2B3FFA6;">allowing messages to be sent and received between applications, even those on different networked machines.</mark> This solves the core problem of how disparate applications, potentially running on different physical machines, can communicate and exchange data over a network.

Their primary application is in building **network servers and clients**, forming the backbone of virtually all internet communication. For instance, <mark style="background: #D2B3FFA6;">every website we visit relies on an HTTP server, which is implemented using sockets. </mark>Famous HTTP servers like Apache Tomcat and [[NginX]] are built on top of TCP sockets.

In computer science and programming, sockets are critically important because they offer a **standardized, low-level interface** for network communication. ==In UNIX-based systems, a socket is conceptually treated like a **file descriptor**, allowing programmers to leverage familiar file I/O operations== (like `read` and `write`) for network interactions. This abstraction simplifies network programming by providing a consistent model for various communication interfaces. Understanding sockets is essential for anyone delving into network programming, cybersecurity, or developing distributed systems.

# Core Explanation:
---

A **socket** is an <mark style="background: #FF5582A6;">endpoint for sending and receiving data across a network. It acts as a programming interface for network communication</mark>. While the socket mechanism is designed to be independent of any specific network type, IP (Internet Protocol) is the most prevalent network technology where sockets are used.

### Key Characteristics:

- **Communication Link**: Sockets establish <mark style="background: #ADCCFFA6;">a communication channel between two processes</mark>, which can be <mark style="background: #ADCCFFA6;">on the same machine or on different machines connected via a network</mark>.
- **File Descriptor Analogy**: In UNIX systems, ==a socket is treated as a **file descriptor**==, a small integer representing an open file or I/O resource, simplifying its manipulation using standard file I/O system calls.

### Types of Sockets:

The <mark style="background: #BBFABBA6;">two main types of sockets relevant to HTTP</mark> are:

1. ==**Stream Sockets== (`SOCK_STREAM`)**:
    - Provide **reliable, two-way, ==connected communication streams**.==
    - ==Use the **Transmission Control Protocol (TCP)**==.
    - ==Prioritize **data quality over speed**==, ensuring data is transferred reliably, without errors, and in the correct sequence (providing flow control and error handling).
    - ==**HTTP primarily uses `SOCK_STREAM`** because it requires a reliable transport layer==.
2. **==Datagram Sockets== (`SOCK_DGRAM`)**:
    - Are **connectionless** sockets.
    - Use the ==**User Datagram Protocol (UDP)*==.
    - ==**Data delivery is not guaranteed*==*; packets may be lost, duplicated, or arrive out of order.
    - Commonly used ==in applications where **speed is a priority over data integrity**==, such as online games, video streaming, and audio communication. **UDP is generally _not_ used for HTTP** due to its unreliability.

### How Sockets Work (TCP Server-Side Programming Flow):

Building a network server (like an HTTP server) typically follows a sequence of system calls:

1. ==**Create the Socket**:==
    >- The `socket()` system call creates a new socket: `int server_fd = socket(domain, type, protocol);`.
    >- **`domain`**: Specifies the communication domain (e.g., `AF_INET` for IPv4).
    >- **`type`**: Specifies the service type (e.g., `SOCK_STREAM` for TCP).
    >- **`protocol`**: Specifies a particular protocol (often `0` for TCP as it's the default for `SOCK_STREAM` with `AF_INET`)
    >- It returns a **file descriptor** (`server_fd`) for the new socket.
2. ==**Identify (Name) the Socket (Binding)**:==
    >- The <mark style="background: #D2B3FFA6;">bind() system call assigns a transport address</mark> (an IP address and port number) to the socket.
    >- `int bind(int socket, const struct sockaddr *address, socklen_t address_len);`.
    >- For IP networking, a `struct sockaddr_in` is used, where `sin_family` (e.g., `AF_INET`), `sin_port` (the port number, typically set using `htons` for network byte order), and `sin_addr` (the IP address, often `INADDR_ANY` for any available interface, converted by `htonl`) are configured.
    >- **`setsockopt()` with `SO_REUSEADDR`** is often used here to allow immediate reuse of a port that was recently closed, preventing "Address already in use" errors.
3. ==**Listen for Incoming Connections**:==
    >- The <mark style="background: #D2B3FFA6;">listen() system call tells the bound socket that it should be capable of accepting incoming connections</mark>: `int listen(int socket, int backlog);`
    >- **`backlog`**: Defines the maximum number of pending connections that can be queued 
4. ==**Accept a Connection**:==
    >- The <mark style="background: #D2B3FFA6;">accept() system call extracts the first connection request from the queue and creates a _new socket_ specifically for that connection</mark> .
    >- `int accept(int socket, struct sockaddr *restrict address, socklen_t *restrict address_len);`  .
    >- The ==_original listening socket remains open_ to accept more connections== .
    >- By default, ==`accept()` is a **blocking operation**==, pausing the program until a connection arrives .
5. ==**Send and Receive Messages**:==
    >- <mark style="background: #FF5582A6;">Once a client and server have connected via their respective sockets, communication occurs</mark> using **`read()` (`recv()`) to receive data and `write()` (`send()`) to send data**  
    >- For an HTTP server, the client sends an HTTP request, and the server processes it to send an HTTP response back, following specific HTTP message formats (headers, blank line, body)
6. ==**Close the Socket**:==
    >- When communication is complete, the `close()` system call is used to terminate the socket connection  

<mark style="background: #BBFABBA6;">Think of it this way: IP is like a building address, TCP is like the mail delivery service, and sockets are like the individual mailboxes that tenants (applications) install to send and receive mail.</mark>
## Webserv Project Relevance

Sockets are at the very **core** of the Webserv project, which challenges participants to build a non-blocking HTTP server from scratch in C++98.

- **Fundamental Communication**: The project requires the server to receive HTTP requests from clients and send back HTTP responses. <mark style="background: #FF5582A6;">Sockets provide the essential communication link for this entire client-server interaction. </mark>Without sockets, there is no network communication for the web server.
- **Non-Blocking Requirement**: A key constraint of the Webserv project is that the server must be **non-blocking at all times** and handle multiple concurrent client connections. This directly necessitates the use of socket programming techniques like `socket()`, `bind()`, `listen()`, and `accept()`, but critically, also requires implementing **non-blocking I/O** using a single polling mechanism like `poll()` (or its equivalents such as `select()` or `epoll()`) for _all_ I/O operations, including listening, reading, and writing. This explicitly forbids blocking calls that would halt the server.
- **Handling HTTP Protocol**: <mark style="background: #ADCCFFA6;">After establishing socket connections, the server must parse incoming data as HTTP requests (e.g., GET, POST, DELETE methods) and construct valid HTTP responses according to RFC specifications</mark> (like RFC 7230-7235 for HTTP/1.1 semantics, though older RFCs are also mentioned as references). <mark style="background: #ADCCFFA6;">All this data exchange happens over the sockets</mark>.
- **Concurrency and Resilience**: The project emphasizes that the server must remain operational under stress, handling client disconnections and large data transfers (like chunked encoding for large POST requests or GET responses) without crashing or hanging indefinitely. Sockets, combined with proper non-blocking I/O and error handling, are the tools to achieve this resilience.
# Related Concepts:
---

- **OSI Model (Open Systems Interconnection Model) & Transport Layer**: The OSI model is a conceptual framework for network communication, divided into seven abstraction layers. HTTP operates at the **Application Layer (Layer 7)**, but it _relies heavily on the **Transport Layer (Layer 4)**_ for reliable data transfer. The Transport Layer, where TCP and UDP operate, is responsible for ensuring data is transferred reliably, in sequence, and with error handling and flow control. Sockets provide the programmatic interface to interact with these Transport Layer protocols.
- **HTTP (Hypertext Transfer Protocol)**: HTTP is an application-level protocol that defines the rules for how web clients (like browsers) and web servers communicate. While HTTP defines the _format and semantics_ of messages (requests and responses), it relies on **sockets** to actually _transport_ those messages over the network. HTTP/1.0 and HTTP/1.1 (the primary focus for building a server from scratch) typically operate over TCP sockets.
- **File Descriptors**: In UNIX-like operating systems, a file descriptor is an abstract indicator (a small integer) used to access I/O resources, such as files, pipes, or network connections. Sockets are treated as file descriptors, meaning that standard I/O operations like `read()`, `write()`, and `close()` can be applied to them, making network programming analogous to file manipulation
- **Blocking vs. Non-Blocking I/O**:
    - **Blocking I/O**: When an I/O operation (like `read()` or `accept()`) is _blocking_, the program's execution pauses or "blocks" until the operation is complete. This is simple to implement for single connections but highly inefficient for servers handling multiple concurrent clients, as one slow operation can halt the entire server.
    - **Non-Blocking / Asynchronous I/O**: Allows a program to initiate an I/O operation and then continue executing other tasks while the I/O operation proceeds in the background. The program is notified when the operation is complete (e.g., data is ready to be read or written). This is crucial for **high-performance web servers** that need to handle many clients simultaneously without being stalled.
- **Polling Functions (`select`, `poll`, `epoll`)**: These are system calls used to implement non-blocking/asynchronous I/O. They allow a single thread to monitor multiple file descriptors (including sockets) to see if they are ready for reading, writing, or have an error, without blocking indefinitely on a single one.
    - **`select()`**: A portable function, but it has limitations, such as a maximum number of file descriptors it can monitor (e.g., 1024 on some Linux systems).
    - **`poll()`**: Often considered a better alternative to `select()` as it overcomes some of `select()`'s limitations regarding the number of file descriptors
    - **`epoll()`**: A Linux-specific and highly scalable alternative, generally superior for very high numbers of connections. These functions allow a server to efficiently manage concurrent connections by only attempting to read from or write to sockets that are known to be ready

# Examples:
---

Here's a simplified C-language example illustrating the basic server-side TCP socket programming steps, inspired by the sources' approach to building an HTTP server from scratch.

```cpp
#include <stdio.h>       // For standard I/O functions like printf, perror [i]
#include <stdlib.h>      // For exit, EXIT_FAILURE [i]
#include <string.h>      // For memset, strlen [i]
#include <sys/socket.h>  // For socket, bind, listen, accept, setsockopt [i]
#include <netinet/in.h>  // For sockaddr_in, INADDR_ANY, htons, htonl [i]
#include <unistd.h>      // For read, write, close [i]

#define PORT 8080        // The port number clients will connect to [i]
#define BUFFER_SIZE 1024 // Size of the buffer for reading data [i]
#define BACKLOG 3        // Max pending connections in the queue for listen() [i]

int main() {
    int server_fd, new_socket;
    struct sockaddr_in address;
    int addrlen = sizeof(address);
    char buffer[BUFFER_SIZE] = {0};

    // Step 1: Create the server socket
    // AF_INET: IPv4 Internet protocols [i]
    // SOCK_STREAM: Provides reliable, two-way, connected communication (TCP) [i]
    // 0: Default protocol for SOCK_STREAM (TCP) [i]
    if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        perror("cannot create socket"); // Print error message if socket creation fails [i]
        exit(EXIT_FAILURE);             // Exit with failure status [i]
    }

    // Optional: Set socket options to allow reuse of address immediately after closing
    // SO_REUSEADDR: Allows other sockets to bind to this port unless there is an active listening socket [i]
    int enable = 1;
    if (setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &enable, sizeof(enable)) < 0) {
        perror("setsockopt(SO_REUSEADDR) failed"); // Print error message if setting option fails [i]
        exit(EXIT_FAILURE);
    }

    // Step 2: Identify (name) the socket (Bind it to an IP address and port)
    // Clear the address structure to ensure no garbage values [i]
    memset(&address, 0, sizeof(address));
    address.sin_family = AF_INET;           // Set address family to IPv4 [i]
    // INADDR_ANY: Allows the server to accept connections on any available network interface [i]
    // htonl(): Converts host byte order to network byte order for IP address [i]
    address.sin_addr.s_addr = htonl(INADDR_ANY);
    // htons(): Converts host byte order to network byte order for port number [i]
    address.sin_port = htons(PORT);         // Set port number [i]

    if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) < 0) {
        perror("bind failed"); // Print error message if bind fails [i]
        exit(EXIT_FAILURE);
    }

    // Step 3: Listen for incoming connections
    // 3: The maximum length to which the queue of pending connections may grow (backlog) [i]
    if (listen(server_fd, BACKLOG) < 0) {
        perror("In listen"); // Print error message if listen fails [i]
        exit(EXIT_FAILURE);
    }

    printf("+++++++ Waiting for new connection ++++++++ on port %d\n", PORT); // Indicate server is ready [i]

    // Step 4: Accept an incoming connection
    // This call blocks until a client connects [i]
    if ((new_socket = accept(server_fd, (struct sockaddr *)&address, (socklen_t*)&addrlen)) < 0) {
        perror("In accept"); // Print error message if accept fails [i]
        exit(EXIT_FAILURE);
    }

    printf("Connection accepted. Reading client request...\n");

    // Read data from the new client socket
    // Reads up to BUFFER_SIZE bytes into 'buffer' [i]
    int valread = read(new_socket, buffer, BUFFER_SIZE);
    if (valread < 0) {
        printf("No bytes are there to read\n"); // If read returns negative, no bytes were read [i]
    } else {
        printf("Client request received (%d bytes):\n%s\n", valread, buffer); // Print the received request [i]
    }

    // Step 5: Send a simple HTTP response
    // This is a minimal HTTP/1.1 200 OK response with plain text content [i]
    // The browser expects this specific format: Status-Line, Headers, blank line, Body [i]
    char *http_response = "HTTP/1.1 200 OK\nContent-Type: text/plain\nContent-Length: 12\n\nHello world!"; // [i]
    // Write the HTTP response back to the client socket [i]
    write(new_socket, http_response, strlen(http_response));
    printf("------------------ 'Hello world!' message sent -------------------\n"); // Confirm message sent [i]

    // Step 6: Close the new client socket
    close(new_socket); // Close the communication socket for the current client [i]

    // Close the listening socket (optional for simple server after one connection)
    // In a real server, this would typically stay open in a loop [i]
    close(server_fd);

    return 0;
}
```

# Flashcards:
---

What is a socket in the context of network programming?;; A socket is a mechanism provided by most operating systems that gives programs access to the network, acting as a communication link between two processes on a network, even those on different networked machines.

List the 5 fundamental steps for TCP socket programming on the server-side. (Excluding non-blocking I/O) ;; 1. Create the socket, 2. Identify (bind) the socket, 3. On the server, wait for an incoming connection (listen), 4. Send and receive messages, 5. Close the socket.

What is the purpose of the `bind()` system call in socket programming?;; The `bind()` system call is used to assign a transport address (an IP address and port number) to a socket. This operation is essential for a server to be reachable by clients at a known address

