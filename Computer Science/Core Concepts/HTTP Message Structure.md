---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-10-10
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
## Purpose/Why:
---

HTTP messages are the fundamental units of communication for the World Wide Web. They solve the problem of ==**standardized communication** between a **client** (like your web browser) and a **server** (the computer hosting a website).==

This standardized structure is crucial because it ensures **interoperability**. Any browser, regardless of who made it (Google, Mozilla, Apple), can communicate with any web server, regardless of the software it runs (Apache, Nginx). This common language is what allows the diverse ecosystem of the internet to function as a cohesive whole. Without this defined structure, the web as we know it couldn't exist. 🌐
## Core Explanation:
---

An **HTTP message** is a block of text data that a client and server exchange. There are two types: **requests** (sent by the client to trigger an action on the server) and **responses** (the answer from the server).

**HTTP Request**

An HTTP request consists of a <mark style="background: #FF5582A6;">request line, headers, and an optional message body</mark>. Here is an example of an HTTP request:

```
GET /index.html HTTP/1.1
Host: localhost:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
```

The<mark style="background: #FF5582A6;"> request line</mark> consists of three parts:<mark style="background: #FF5582A6;"> the method, the path, and the HTTP version.</mark> The method specifies the action that the client wants to perform, such as GET (to retrieve a resource) or POST (to submit data to the server). ==The path or [[URI - Similarities and Differences with URL]] specifies the location of the resource on the server.== The <mark style="background: #FF5582A6;">HTTP version indicates the version of the HTTP protocol being used.</mark>

<mark style="background: #ADCCFFA6;">Headers contain additional information about the request, such as the hostname of the server, and the type of browser being used.</mark>

In the example above there was no message body because GET method usually doesn't include any body.

**HTTP Response**

An <mark style="background: #BBFABBA6;">HTTP response also consists of a status line, headers, and an optional message body.</mark> Here is an example of an HTTP response:

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1234

<Message Body>
```

The <mark style="background: #BBFABBA6;">status line</mark> consists of three parts:<mark style="background: #BBFABBA6;"> the HTTP version, the status code, and the reason phrase</mark>. The <mark style="background: #BBFABBA6;">status code</mark> indicates the <mark style="background: #BBFABBA6;">result of the request, such as 200 OK (successful) or 404 Not Found (resource not found)</mark>. The <mark style="background: #BBFABBA6;">reason phrase is a short description of the status code.</mark> Following is a very brief summary of what a status code denotes:

`1xx` indicates an informational message only

`2xx` indicates success of some kind

`3xx` redirects the client to another URL

`4xx` indicates an error on the client's part

`5xx` indicates an error on the server's part

Headers contain additional information about the response, such as the type and size of the content being returned. The message body contains the actual content of the response, such as the HTML code for a webpage.

**[[HTTP Methods]]**

|Method|Description|Possible Body|
|:--|---|:-:|
|**`GET`**|Retrieve a specific resource or a collection of resources, should not affect the data/resource|No|
|**`POST`**|Perform resource-specific processing on the request content|Yes|
|**`DELETE`**|Removes target resource given by a URI|Yes|
|**`PUT`**|Creates a new resource with data from message body, if resource already exist, update it with data in body|Yes|
|**`HEAD`**|Same as GET, but do not transfer the response content|No|

**GET**

HTTP GET method is used to read (or retrieve) a representation of a resource. In case of success (or non-error), GET returns a representation of the resource in response body and HTTP response status code of 200 (OK). In an error case, it most often returns a 404 (NOT FOUND) or 400 (BAD REQUEST).

**POST**

HTTP POST method is most often utilized to create new resources. On successful creation, HTTP response code 201 (Created) is returned.

**DELETE**

HTTP DELETE is stright forward. It deletes a resource specified in URI. On successful deletion, it returns HTTP response status code 204 (No Content).

Both message types share a similar structure, consisting of three main parts:

1. **Start-Line**: The first line of the message.
    
    - For a **Request**: Contains the HTTP method (e.g., `GET`, `POST`), the target resource path (URI), and the HTTP version. `GET /pages/index.html HTTP/1.1`
        
    - For a **Response**: Contains the HTTP version, a numerical status code, and a reason phrase. `HTTP/1.1 200 OK`
        
2. **Headers**: A set of key-value pairs that provide metadata about the message.
    
    - Headers provide context, such as the host, the format of the message body (`Content-Type`), browser information (`User-Agent`), and caching instructions.
        
    - Example: `Content-Type: application/json`
        
3. **Blank Line**: A single, mandatory blank line (a carriage return followed by a line feed, or `CRLF`) that separates the headers from the body. This is the signal that the metadata section has ended.
    
4. **Body (Optional)**: Contains the actual data of the message.
    
    - Its presence depends on the request/response type. For example, a `GET` request typically has no body, while a `POST` request (like submitting a form) does.
        
    - A successful `200 OK` response to a `GET` request will have a body containing the requested resource (e.g., HTML, CSS, JSON).
        
## Related Concepts:
---

- **[[TCP]]/[[IP (Internet Protocol)]]**: HTTP is an application layer protocol that typically runs on top of TCP/IP. TCP (Transmission Control Protocol) is responsible for breaking the HTTP message into packets, ensuring they are delivered reliably and in the correct order, and reassembling them on the other end.
    
- **HTTP Methods**: The action specified in the request's start-line. Common methods include `GET` (retrieve data), `POST` (submit data), `PUT` (update data), and `DELETE` (remove data).
    
- **HTTP Status Codes**: The three-digit codes in the response's start-line that indicate the result of the request. They are grouped into categories: `1xx` (Informational), `2xx` (Success), `3xx` (Redirection), `4xx` (Client Error), and `5xx` (Server Error).
    
- **URI/URL**: A Uniform Resource Identifier is used in the request's start-line to specify the resource (e.g., a specific webpage, API endpoint, or file) that the client is targeting.
    
## Examples:

---

Here are raw text examples of HTTP messages. Note that these are text-based and readable.

```http
# --- HTTP GET Request ---
# A client asking the server for the main page of a website.

# 1. Start-Line: Method is GET, resource is '/', version is HTTP/1.1
GET /index.html HTTP/1.1

# 2. Headers: Key-value pairs providing metadata about the request.
Host: www.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: text/html,*/*
Accept-Language: en-US,en;q=0.5

# 3. Blank Line: A mandatory CRLF separates headers from the (empty) body.

# 4. Body: Empty, because GET requests don't need to send data.
```

```http
# --- HTTP POST Request ---
# A client sending new user data to an API endpoint.

# 1. Start-Line: Method is POST, resource is '/api/users', version is HTTP/1.1
POST /api/users HTTP/1.1

# 2. Headers: Note the Content-Type and Content-Length, which describe the body.
Host: api.example.com
Content-Type: application/json
Content-Length: 39

# 3. Blank Line: Separates headers from the body.

# 4. Body: Contains the data being sent, in JSON format.
{
  "username": "testuser",
  "email": "test@example.com"
}
```

```http
# --- HTTP Response ---
# The server's successful response to the GET request above.

# 1. Start-Line: Version, Status Code (200 = Success), and Reason Phrase.
HTTP/1.1 200 OK

# 2. Headers: Metadata about the response. Content-Type tells the browser how to render the body.
Date: Wed, 06 Aug 2025 13:50:35 GMT
Server: Apache/2.4.1 (Unix)
Content-Type: text/html; charset=UTF-8
Content-Length: 1270

# 3. Blank Line: Separates headers from the body.

# 4. Body: The actual HTML content of the page.
<!DOCTYPE html>
<html>
<head>
  <title>Example Page</title>
</head>
<body>
  <h1>Hello, World!</h1>
  <p>This is a test page.</p>
</body>
</html>
```

## Flashcards:
---

What are the two main types of HTTP messages?;;Requests (from client to server) and Responses (from server to client).

What are the three mandatory parts of an HTTP message's structure?;;Start-Line, Headers, and a separating Blank Line. The Body is optional.

What is the purpose of the blank line in an HTTP message?;;It's a mandatory separator (CRLF) that signals the end of the headers and the beginning of the message body.

What three pieces of information are in a request's start-line?;;The HTTP Method (e.g., GET), the resource URI (e.g., /index.html), and the HTTP Version (e.g., HTTP/1.1).

What three pieces of information are in a response's start-line?;;The HTTP Version, a numeric Status Code (e.g., 200 or 404), and a text Status Phrase (e.g., OK or Not Found).

What is the function of HTTP headers?;;To provide metadata about the request or response in the form of key-value pairs (e.g., Content-Type: application/json).