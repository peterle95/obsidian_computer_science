---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-10-02
scheda: done
visit-count: 4
confidence-level: 2.5
consecutive-correct: 3
last-struggle-date: 2025-08-12
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

The Internet Protocol (IP) solves the fundamental problem of **routing data across interconnected networks**. <mark style="background: #FF5582A6;">Without IP, computers on different networks couldn't communicate with each other</mark> - data would have <mark style="background: #FF5582A6;">no way to find its destination across the complex web of routers, switches, and networks that make up the internet</mark>.

IP is critical because it:

- **Enables global connectivity**: Allows any device on the internet to communicate with any other device
- **Provides addressing**: Each device gets a unique IP address, like a postal address for data packets
- **Handles routing**: Determines the best path for data to travel from source to destination
- **Works with higher-level protocols**: Forms the foundation that protocols like TCP, UDP, and HTTP build upon

In the context of web servers and the webserv project, IP is essential because it's how clients (web browsers) locate and connect to your server across the network.

# **Core Explanation:**
---

**Internet Protocol (IP)** is a network layer protocol that provides addressing and routing services for data transmission across interconnected networks. It operates at Layer 3 (Network Layer) of the OSI model.

**Key Characteristics:**

- **Connectionless**: Each packet is treated independently, no connection state maintained
- **Best-effort delivery**: No guarantee of delivery, ordering, or data integrity
- **Packet-based**: Data is broken into discrete packets for transmission
- **Hierarchical addressing**: IP addresses have network and host portions for efficient routing

**How IP Works:**

1. **Addressing**: Every device gets a unique IP address (e.g., 192.168.1.100)
2. **Packetization**: Data is divided into IP packets, each with source/destination addresses
3. **Routing**: Routers examine destination addresses and forward packets toward their destination
4. **Reassembly**: Destination device collects packets and reassembles original data

**IP Packet Structure:**

- **Header**: Contains routing information (source IP, destination IP, packet length, etc.)
- **Payload**: The actual data being transmitted

**IP Versions:**

- **IPv4**: 32-bit addresses (e.g., 192.168.1.1), supports ~4.3 billion addresses
- **IPv6**: 128-bit addresses, designed to solve IPv4 address exhaustion


# **Related Concepts:**
---

**[[TCP]] (Transmission Control Protocol)**:

- **Relationship**: <mark style="background: #FF5582A6;">TCP runs on top of IP (TCP/IP stack)</mark>
- **Difference**: <mark style="background: #ADCCFFA6;">IP handles routing and addressing; TCP adds reliability, ordering, and connection management</mark>
- **Together**: <mark style="background: #D2B3FFA6;">IP delivers packets; TCP ensures they arrive correctly and in order</mark>

**UDP (User Datagram Protocol)**:

- **Relationship**: Alternative transport protocol that also runs on IP
- **Difference**: Simpler than TCP, no reliability guarantees, faster for real-time applications

**HTTP (Hypertext Transfer Protocol)**:

- **Relationship**: Application layer protocol that runs on top of TCP/IP
- **Connection**: <mark style="background: #FF5582A6;">Web browsers use HTTP over TCP over IP to communicate with web servers</mark>

**[[Sockets]]**:

- **Relationship**: <mark style="background: #BBFABBA6;">Programming interface that allows applications to use TCP/IP</mark>
- **Function**: <mark style="background: #BBFABBA6;">Sockets combine IP addresses with port numbers for complete endpoint identification</mark>

**DNS (Domain Name System)**:

- **Relationship**: Translates human-readable domain names to IP addresses
- **Purpose**: Makes IP addressing user-friendly (google.com → 142.250.191.14)

**Routing**:

- **Relationship**: The process IP uses to determine packet paths
- **Mechanism**: Routers maintain routing tables to make forwarding decisions

## **Application in Webserv Project:**
---

In the webserv project, IP plays several crucial roles:

**1. Server Binding and Listening:** The configuration file allows specifying multiple IP addresses and ports where the server should listen. Each combination creates a unique endpoint that clients can connect to.

**2. Client Identification:** When handling HTTP requests, the server needs to identify which client sent the request. The client's IP address is extracted from the socket connection and can be used for logging, access control, or rate limiting.

**3. Multiple Virtual Hosts:** Although virtual hosts are mentioned as out-of-scope, the server still needs to handle multiple IP/port combinations, each potentially serving different content based on the configuration.

**4. Network I/O Management:** The non-blocking I/O requirement means the server must efficiently handle multiple IP connections simultaneously using `poll()`, `select()`, or equivalent functions. Each connection is identified by its socket file descriptor, which corresponds to a specific IP connection.

**5. CGI Environment Variables:** When executing CGI scripts, the server must provide client IP information through environment variables like `REMOTE_ADDR`, which contains the client's IP address.

**Example webserv configuration handling:**

```cpp
// In webserv, you might parse configuration like:
// listen 127.0.0.1:8080;
// listen 0.0.0.0:80;
// This creates two server sockets bound to different IP addresses
// allowing the same server process to serve different content
// based on which IP address the client connects to
```

The webserv project essentially creates a bridge between the low-level IP networking (sockets, addresses, connections) and high-level HTTP protocol handling (parsing requests, generating responses).

# **Examples:**
---

```cpp
// Example 1: Basic socket creation and IP address handling in C++
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <iostream>

// Creating a socket that will use IP for communication
int server_socket = socket(AF_INET, SOCK_STREAM, 0);
// AF_INET specifies IPv4 protocol family
// SOCK_STREAM means we want TCP on top of IP
// 0 lets the system choose the appropriate protocol (TCP for SOCK_STREAM)

// Setting up server address structure - this defines the IP endpoint
struct sockaddr_in server_addr;
server_addr.sin_family = AF_INET;          // Use IPv4
server_addr.sin_addr.s_addr = INADDR_ANY;  // Listen on all available IP interfaces
server_addr.sin_port = htons(8080);        // Port 8080, converted to network byte order

// Bind socket to specific IP address and port
// This tells the OS "when IP packets arrive for this address/port, give them to our program"
bind(server_socket, (struct sockaddr*)&server_addr, sizeof(server_addr));

// Example 2: IP address conversion and validation
std::string ip_str = "192.168.1.100";
struct in_addr addr;

// Convert string IP to binary format (what IP actually uses internally)
if (inet_pton(AF_INET, ip_str.c_str(), &addr) == 1) {
    std::cout << "Valid IPv4 address" << std::endl;
    
    // Convert back to string format for display
    char buffer[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &addr, buffer, INET_ADDRSTRLEN);
    std::cout << "Binary representation converted back: " << buffer << std::endl;
}

// Example 3: Getting client IP information (useful in webserv)
struct sockaddr_in client_addr;
socklen_t client_len = sizeof(client_addr);

// Accept connection and get client's IP information
int client_socket = accept(server_socket, (struct sockaddr*)&client_addr, &client_len);

// Extract client's IP address from the connection
char client_ip[INET_ADDRSTRLEN];
inet_ntop(AF_INET, &client_addr.sin_addr, client_ip, INET_ADDRSTRLEN);
std::cout << "Client connected from IP: " << client_ip << std::endl;
std::cout << "Client port: " << ntohs(client_addr.sin_port) << std::endl;
// ntohs converts from network byte order to host byte order
```

```cpp
// Example 4: Webserv-specific IP handling for multiple interfaces
class WebServer {
private:
    std::vector<int> server_sockets;  // Multiple sockets for different IP/port combinations
    
public:
    // Configure server to listen on multiple IP addresses and ports
    void setupListeners(const std::vector<std::string>& ips, const std::vector<int>& ports) {
        for (size_t i = 0; i < ips.size(); ++i) {
            int sock = socket(AF_INET, SOCK_STREAM, 0);
            
            struct sockaddr_in addr;
            addr.sin_family = AF_INET;
            addr.sin_port = htons(ports[i]);
            
            // Convert IP string to binary format
            if (inet_pton(AF_INET, ips[i].c_str(), &addr.sin_addr) <= 0) {
                // Handle invalid IP address
                std::cerr << "Invalid IP address: " << ips[i] << std::endl;
                continue;
            }
            
            // Bind to specific IP and port
            if (bind(sock, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
                std::cerr << "Failed to bind to " << ips[i] << ":" << ports[i] << std::endl;
                continue;
            }
            
            listen(sock, 128);  // Start listening for connections
            server_sockets.push_back(sock);
            
            std::cout << "Server listening on " << ips[i] << ":" << ports[i] << std::endl;
        }
    }
};

// Usage in webserv configuration
WebServer server;
std::vector<std::string> listen_ips = {"127.0.0.1", "0.0.0.0", "192.168.1.10"};
std::vector<int> listen_ports = {8080, 80, 3000};
server.setupListeners(listen_ips, listen_ports);
// This allows the server to accept connections on multiple IP/port combinations
// 127.0.0.1:8080 - localhost only
// 0.0.0.0:80 - all available interfaces
// 192.168.1.10:3000 - specific network interface
```

# **Flashcards:**
---

What is the primary purpose of the Internet Protocol (IP)?;; IP provides addressing and routing services to enable data communication across interconnected networks, allowing devices on different networks to find and communicate with each other.

What are the key differences between IP and TCP in the TCP/IP stack?;; IP handles addressing and routing (getting packets to the right destination), while TCP adds reliability, connection management, and ensures data arrives correctly and in order. IP is connectionless; TCP is connection-oriented.

What information is contained in an IP packet header?;; An IP packet header contains source IP address, destination IP address, packet length, time-to-live (TTL), protocol type, checksum, and other routing information needed to deliver the packet.

How does IP addressing enable global internet connectivity?;; IP addresses provide unique identifiers for devices (like postal addresses), allowing routers to make forwarding decisions and find optimal paths for packets to reach their destinations across the complex internet infrastructure.

In the webserv project, why is IP important for handling multiple server configurations?;; IP allows webserv to bind to multiple IP addresses and ports simultaneously, enabling the server to serve different content based on which IP/port combination clients connect to, supporting multiple virtual hosts or service endpoints.

What is the relationship between IP addresses and socket programming in webserv?;; Sockets combine IP addresses with port numbers to create complete network endpoints. In webserv, you bind sockets to specific IP/port combinations to listen for connections, and extract client IP addresses from accepted connections for logging and request handling.