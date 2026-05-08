---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-09-16
scheda: done
visit-count: 5
confidence-level: 3
consecutive-correct: 4
last-struggle-date: 2025-08-07
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
**TCP (Transmission Control Protocol)** solves the fundamental problem of **reliable and ordered delivery of data** ==across an unreliable network like the internet==. Imagine sending a multi-page document through the mail where pages can get lost, arrive out of order, or even arrive multiple times. TCP acts like a sophisticated mail service that ensures all pages arrive, in the correct order, and only once, even if the underlying network is chaotic.

It's crucial in computer science because it <mark style="background: #FFB86CA6;">underpins the vast majority of internet applications we use daily, from web Browse (HTTP) and email (SMTP) to file transfer (FTP) and secure shell (SSH). </mark>Without TCP, building applications that require dependable data exchange would be incredibly complex, as every developer would have to implement their own mechanisms for error checking, retransmissions, and ordering. Its importance lies in providing a robust and standardized way to achieve **end-to-end reliable communication**.

# **Core Explanation:**
---
TCP is a **connection-oriented protocol** that operates at the ==transport layer of the internet's network stack==. It provides a **reliable, ordered, and error-checked delivery of a stream of bytes** between applications running on hosts.

Key characteristics include:

- **Connection-Oriented:** ==Before data can be exchanged, a connection must be established== between the sender and receiver using a "<mark style="background: #FF5582A6;">three-way handshake</mark>." <mark style="background: #FF5582A6;">This handshake involves SYN (synchronize), SYN-ACK (synchronize-acknowledgment), and ACK (acknowledgment) packets to establish initial sequence numbers and ensure both sides are ready to communicate</mark>.
    
- **Reliable Data Transfer:** TCP guarantees that data sent will be received by the other end. It achieves this through **acknowledgments (ACKs)** for received data segments and **retransmission timeouts**. If an ACK isn't received within a certain time, the sender retransmits the data.
    
- **Ordered Data Delivery:** <mark style="background: #ADCCFFA6;">Data is delivered to the application in the same order it was sent</mark>. If segments arrive out of order, TCP buffers them until the missing segments arrive and then reassembles them in the correct sequence.
    
- **Flow Control:** <mark style="background: #FFB8EBA6;">TCP prevents a fast sender from overwhelming a slow receiver.</mark> It uses a **sliding window mechanism**, where the receiver advertises a "window size" indicating how much buffer space it has available. The sender will not send more data than the receiver can currently handle.
    
- **Congestion Control:** TCP aims to prevent network congestion. It dynamically adjusts the rate at which data is sent based on perceived network conditions. Mechanisms like "slow start," "congestion avoidance," and "fast retransmit/recovery" are employed to manage the amount of data injected into the network.
    
- **Full-Duplex Communication:** <mark style="background: #FF5582A6;">Data can be sent and received simultaneously over a single TCP connection.</mark>
    

In essence, TCP takes an application's data, breaks it into segments, adds a header with control information (like sequence numbers and acknowledgment numbers), and passes it down to the IP layer for transmission. At the receiving end, it reassembles the segments, acknowledges their receipt, and delivers the ordered stream of data to the receiving application.

# **Related Concepts:**
---
- **UDP (User Datagram Protocol):** Like TCP, UDP operates at the transport layer. However, unlike TCP, UDP is a **connectionless and unreliable protocol**. It sends data packets (datagrams) without establishing a prior connection, acknowledgments, or guarantees of delivery, order, or error-checking. UDP is faster and has less overhead than TCP, making it suitable for applications where speed is more critical than reliability, such as streaming video, online gaming, and DNS lookups. TCP builds reliability on top of the best-effort delivery of IP, while UDP simply passes data directly to IP.


<img src="assets/images/udp-tcp.jpg" alt="JavaScript Garbage Collection example" style="width: 750px; height: auto;" />
    
- **[[IP (Internet Protocol)]]:** IP operates at the network layer, beneath TCP and UDP. It's responsible for **addressing and routing packets** across the internet. IP provides a "best-effort" delivery service, meaning it tries to deliver packets but offers no guarantees of delivery, order, or error-checking. TCP and UDP rely on IP to move their segments/datagrams between hosts. TCP adds reliability and other features on top of IP's basic delivery service.
    
- **[[Sockets]]:** Sockets are an **API (Application Programming Interface)** that applications use to access the underlying network protocols like TCP or UDP. They provide a programming interface for network communication, abstracting away the complexities of the protocol stack. When you write code to establish a TCP connection, you're typically using socket programming functions (e.g., `socket()`, `connect()`, `bind()`, `listen()`, `accept()`, `send()`, `recv()`).
    
- **OSI Model / TCP/IP Model:** TCP is a key component of the **TCP/IP model**, specifically residing at the **Transport Layer**. The OSI (Open Systems Interconnection) model is a conceptual framework that standardizes communication functions of a telecommunication or computing system. Understanding these models helps to place TCP within the broader context of network communication and understand its role relative to other protocols and layers.
    

# **Examples:**
---

```python
# Python example demonstrating a basic TCP client-server interaction
# This code illustrates the core concepts of establishing a TCP connection
# and sending/receiving data reliably.

# --- Server Side (server.py) ---
import socket

HOST = '127.0.0.1'  # Standard loopback interface address (localhost)
PORT = 65432        # Port to listen on (non-privileged ports are > 1023)

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    # socket.socket() creates a socket object.
    # AF_INET specifies the address family (IPv4).
    # SOCK_STREAM specifies the socket type (TCP).

    s.bind((HOST, PORT)) # Binds the socket to the host address and port.
                         # This makes the server listen on this specific network interface and port.

    s.listen()           # Enables the server to accept incoming connections.
                         # The argument (optional, typically 1 or 5) specifies the maximum number of queued connections.

    conn, addr = s.accept() # Blocks and waits for an incoming connection.
                            # When a client connects, it returns a new socket object 'conn'
                            # representing the connection to the client, and 'addr'
                            # which is the address of the client.

    with conn:
        print(f"Connected by {addr}")
        while True:
            data = conn.recv(1024) # Receives up to 1024 bytes of data from the client.
                                  # This call blocks until data is received.
            if not data:
                break # If no data is received (client closed connection), break the loop.
            print(f"Received from client: {data.decode()}")
            conn.sendall(b"Echo: " + data) # Sends the received data back to the client.
                                          # b"" indicates a byte string.
                                          # sendall() ensures all data is sent.

print("Server closed connection.")

# --- Client Side (client.py) ---
import socket

HOST = '127.0.0.1'  # The server's hostname or IP address
PORT = 65432        # The port used by the server

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.connect((HOST, PORT)) # Establishes a connection to the server.
                            # This initiates the TCP three-way handshake.

    message = "Hello, TCP!"
    s.sendall(message.encode()) # Sends data to the server.
                                # .encode() converts the string to bytes.

    data = s.recv(1024)         # Receives up to 1024 bytes of data from the server.

print(f"Received from server: {data.decode()}")

# How to run these examples:
# 1. Save the server code as `server.py` and the client code as `client.py` in the same directory.
# 2. Open two separate terminal windows.
# 3. In the first terminal, run the server: `python server.py`
# 4. In the second terminal, run the client: `python client.py`
# You will see the "Connected by..." message on the server terminal and the "Received from server..." message on the client terminal.
```

# **Flashcards:**
---

What problem does TCP solve?;;TCP solves the problem of reliable, ordered, and error-checked delivery of data across an unreliable network.

What are the three main characteristics of TCP?;;Connection-oriented, reliable data transfer, and flow/congestion control.

How does TCP ensure reliable data transfer?;;Through acknowledgments (ACKs) and retransmission timeouts for unacknowledged data segments.