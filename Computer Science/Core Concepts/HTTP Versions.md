---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-10-18
scheda: done
visit-count: 6
confidence-level: 3.5
consecutive-correct: 5
last-struggle-date: 2025-08-15
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
# **Purpose/Why**:
---

HTTP versions solve the fundamental problem of standardizing communication between web clients and servers over time. As the web evolved, different versions of HTTP addressed performance, security, and feature limitations. ==HTTP versions define the rules for request/response formatting, connection handling, header structure, and data transfer methods==. This standardization enables interoperability between different browsers, servers, and web applications worldwide, ensuring that a client can communicate with any compliant server regardless of the underlying implementation.

# **Core Explanation:**
---

HTTP (Hypertext Transfer Protocol) has evolved through several major versions:

**HTTP/0.9 (1991)**: The original protocol supporting only GET requests for HTML documents. No headers, status codes, or error handling.

**HTTP/1.0 (1996)**: Introduced headers, status codes, POST method, and support for different content types. <mark style="background: #FF5582A6;">Each request required a new TCP connection,</mark> making it inefficient for multiple resources.

**HTTP/1.1 (1997)**: Added<mark style="background: #BBFABBA6;"> persistent connections (keep-alive)</mark>, <mark style="background: #BBFABBA6;">chunked transfer encoding, additional methods (PUT, DELETE, PATCH), host headers enabling virtual hosting, and caching improvements.</mark> This became the dominant version for decades.

**HTTP/2 (2015)**: Binary protocol with multiplexing, server push, header compression, and stream prioritization. Maintains backward compatibility with HTTP/1.1 semantics.

**HTTP/3 (2022)**: ==Uses [[QUIC]] instead of TCP, providing better performance over unreliable networks with built-in encryption and reduced connection setup time.==

Each version maintains backward compatibility while adding new features to improve performance, security, and functionality.

## **Project Requirements (Webserv)**

For this project, **HTTP/1.0 is suggested as a reference point** but not strictly enforced. The key requirements are:

- Support GET, POST, and DELETE methods
- Handle standard HTTP headers and status codes
- Implement [[Non-blocking I⧸O]] with proper connection management
- Compatibility with standard web browsers
- Comparison with NGINX behavior for reference

# **Related Concepts:**
---

**[[TCP]]/[[IP (Internet Protocol)]]**: HTTP operates at the application layer above TCP, which provides reliable connection-oriented communication. HTTP versions 1.0-2 use TCP, while HTTP/3 uses QUIC.

**Request/Response Cycle**: All HTTP versions follow the basic client-server request/response pattern, though the underlying implementation varies.

**Status Codes**: Standardized across all HTTP versions (200 OK, 404 Not Found, 500 Internal Server Error) to indicate request outcomes.

**[[HTTP Headers]]**: Metadata accompanying HTTP messages. HTTP/1.0 introduced them, HTTP/1.1 expanded them, and HTTP/2 compressed them.

**Connection Management**: Evolution from HTTP/1.0's connection-per-request to HTTP/1.1's persistent connections to HTTP/2's multiplexing.

**[[Common Gateway Interface (CGI)]]**: A standard for web servers to execute external programs, independent of HTTP version but affected by how the server handles the underlying HTTP communication.

# **Examples:**
---

```cpp
// HTTP/1.0 Request Example - Simple GET request
// Note: Each request requires a new TCP connection
/*
GET /index.html HTTP/1.0
Host: www.example.com
User-Agent: MyWebServer/1.0
Connection: close

// No request body for GET
*/

// HTTP/1.0 Response Example
/*
HTTP/1.0 200 OK
Date: Wed, 06 Aug 2025 10:00:00 GMT
Server: webserv/1.0
Content-Type: text/html
Content-Length: 1234
Connection: close

<html>
<head><title>Example Page</title></head>
<body><h1>Hello World</h1></body>
</html>
*/
```

```cpp
// HTTP/1.1 Request Example - With persistent connection
// Connection can be reused for multiple requests
/*
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: MyWebServer/1.0
Connection: keep-alive
Accept: text/html,application/xhtml+xml
Accept-Encoding: gzip, deflate

// Empty line indicates end of headers
*/

// HTTP/1.1 Response with chunked encoding
/*
HTTP/1.1 200 OK
Date: Wed, 06 Aug 2025 10:00:00 GMT
Server: webserv/1.0
Content-Type: text/html
Transfer-Encoding: chunked
Connection: keep-alive

1A4
<html><head><title>Chunked Response</title></head>
<body><h1>This is chunked data</h1></body></html>
0

// Connection remains open for next request
*/
```

```cpp
// POST Request Example - File upload capability required by project
/*
POST /upload HTTP/1.1
Host: www.example.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Length: 345
Connection: keep-alive

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="file"; filename="example.txt"
Content-Type: text/plain

File content goes here...
------WebKitFormBoundary7MA4YWxkTrZu0gW--
*/

// DELETE Request Example - Required method for project
/*
DELETE /files/example.txt HTTP/1.1
Host: www.example.com
Authorization: Bearer token123
Content-Length: 0
Connection: keep-alive

// No body for DELETE request
*/
```

```cpp
// Error Response Examples - Project requires accurate status codes
/*
// 404 Not Found
HTTP/1.1 404 Not Found
Date: Wed, 06 Aug 2025 10:00:00 GMT
Server: webserv/1.0
Content-Type: text/html
Content-Length: 123
Connection: close

<html><body><h1>404 - Page Not Found</h1></body></html>

// 500 Internal Server Error
HTTP/1.1 500 Internal Server Error
Date: Wed, 06 Aug 2025 10:00:00 GMT
Server: webserv/1.0
Content-Type: text/html
Content-Length: 145
Connection: close

<html><body><h1>500 - Internal Server Error</h1></body></html>
*/
```

# **Flashcards:**
---

What are the main differences between HTTP/1.0 and HTTP/1.1?;; HTTP/1.0 requires a new TCP connection for each request and has limited header support. HTTP/1.1 introduces persistent connections (keep-alive), chunked transfer encoding, additional HTTP methods (PUT, DELETE), host headers for virtual hosting, and improved caching mechanisms.

Which HTTP version is recommended for the webserv project and what methods must be supported?;; HTTP/1.0 is suggested as a reference point but not enforced. The project must support at least GET, POST, and DELETE methods, handle standard headers and status codes, and be compatible with web browsers.

What is the purpose of the Connection header in HTTP requests?;; The Connection header controls whether the network connection stays open after the current transaction. "Connection: close" closes the connection after response, while "Connection: keep-alive" maintains the connection for reuse in subsequent requests.

What is chunked transfer encoding and in which HTTP version was it introduced?;; Chunked transfer encoding was introduced in HTTP/1.1. It allows sending data in chunks without knowing the total content length beforehand. Each chunk is prefixed with its size in hexadecimal, ending with a zero-sized chunk to indicate completion.

What are HTTP status codes and give examples of the main categories?;; HTTP status codes are three-digit numbers indicating the result of an HTTP request. Main categories: 2xx (success, e.g., 200 OK), 3xx (redirection, e.g., 301 Moved Permanently), 4xx (client error, e.g., 404 Not Found), 5xx (server error, e.g., 500 Internal Server Error).

How does HTTP/2 differ from HTTP/1.1 in terms of performance and connection handling?;; HTTP/2 is a binary protocol that supports multiplexing (multiple requests over single connection), server push, header compression, and stream prioritization. Unlike HTTP/1.1's text-based format and sequential request handling, HTTP/2 allows parallel processing while maintaining semantic compatibility.