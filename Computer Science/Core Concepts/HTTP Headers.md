---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-10-22
scheda: done
visit-count: 7
confidence-level: 3
consecutive-correct: 4
last-struggle-date: 2025-08-30
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

HTTP Headers solve the fundamental problem of <mark style="background: #BBFABBA6;">providing metadata and control information for HTTP requests and responses.</mark> They enable clients and servers <mark style="background: #BBFABBA6;">to communicate essential details about the data being transmitted, authentication requirements, caching policies, content types, and connection preferences</mark>. Headers are crucial because they <mark style="background: #BBFABBA6;">allow HTTP to be a stateless yet flexible protocol</mark> - each request/response can carry all the context needed for proper handling without maintaining persistent connections or server-side state.

# **Core Explanation:**
---

HTTP Headers are key-value pairs sent <mark style="background: #FF5582A6;">at the beginning of HTTP requests and responses</mark>, <mark style="background: #FF5582A6;">before the actual message body.</mark> They provide metadata about the HTTP transaction, including information about the content, the client, the server, and how the message should be processed. Headers are case-insensitive and follow the format "Header-Name: Header-Value". They are categorized into four types: General headers (apply to both requests and responses), Request headers (specific to requests), Response headers (specific to responses), and Entity headers (describe the message body). <mark style="background: #BBFABBA6;">Headers enable functionality like content negotiation, authentication, caching, cookies, CORS, compression, and connection management.</mark> They are transmitted in plain text and terminated by a blank line before the message body begins.


<img src="assets/images/HTTP Header.jpeg" style="width: 450px; height: auto;" />

# **Related Concepts:**
---

- HTTP Protocol - Headers are an integral part of the HTTP specification and enable the protocol's flexibility. 
- REST APIs - Headers are essential for RESTful services to handle authentication, content types, and API versioning. 
- [[CORS (Cross-Origin Resource Sharing)]] - Uses specific headers to manage cross-domain requests. 
- Authentication - Headers like Authorization carry credentials and tokens. Content Negotiation - Headers enable clients and servers to agree on data formats, languages, and encodings. 
- Caching ([[Cache]]) - Cache-Control and related headers manage browser and proxy caching behavior. 
- Cookies - Set-Cookie and Cookie headers manage session state. 
- WebSockets - Connection and Upgrade headers facilitate protocol switching._

# **Examples:**
---

```javascript
// Example 1: Making a fetch request with custom headers
fetch('https://api.example.com/users', {
  method: 'GET',
  headers: {
    'Content-Type': 'application/json',        // Specifies the media type of the request body
    'Authorization': 'Bearer eyJhbGciOiJIUzI1NiIs...', // Carries authentication credentials
    'Accept': 'application/json',              // Tells server what content types client can process
    'User-Agent': 'MyApp/1.0',                // Identifies the client application
    'Accept-Language': 'en-US,en;q=0.9',      // Specifies preferred languages for response
  }
})
.then(response => {
  // Accessing response headers
  console.log('Content-Type:', response.headers.get('Content-Type')); // Gets the response content type
  console.log('Cache-Control:', response.headers.get('Cache-Control')); // Gets caching directives
  return response.json();
});

// Example 2: Express.js server setting response headers
const express = require('express');
const app = express();

app.get('/api/data', (req, res) => {
  // Reading request headers sent by client
  const userAgent = req.headers['user-agent'];     // Client's browser/app info
  const acceptLanguage = req.headers['accept-language']; // Client's language preferences
  const authorization = req.headers.authorization;  // Authentication token if present
  
  // Setting response headers to control client behavior
  res.set({
    'Content-Type': 'application/json',             // Tells client the response format
    'Cache-Control': 'max-age=3600, public',       // Allows caching for 1 hour
    'Access-Control-Allow-Origin': '*',            // CORS header for cross-domain access
    'X-API-Version': '1.2.3',                      // Custom header with API version info
    'X-Rate-Limit-Remaining': '99'                 // Custom header showing remaining API calls
  });
  
  res.json({ message: 'Hello World', userAgent: userAgent });
});

// Example 3: Common HTTP headers in a raw HTTP request/response
/*
Raw HTTP Request:
GET /api/users HTTP/1.1
Host: example.com                          // Specifies the target server
Accept: application/json, text/html        // What content types client accepts  
Accept-Encoding: gzip, deflate             // What compression methods client supports
Connection: keep-alive                     // Requests persistent connection
Cookie: sessionId=abc123; theme=dark       // Sends stored cookies to server
If-None-Match: "etag123"                   // Conditional request for caching

Raw HTTP Response:
HTTP/1.1 200 OK
Content-Type: application/json             // Format of the response body
Content-Length: 156                        // Size of response body in bytes
Set-Cookie: sessionId=xyz789; HttpOnly     // Sets a cookie on client browser
ETag: "etag456"                           // Version identifier for caching
Last-Modified: Wed, 21 Oct 2023 07:28:00 GMT // When resource was last changed
Server: nginx/1.18.0                      // Information about server software
*/

// Example 4: Security-related headers
app.use((req, res, next) => {
  // Setting security headers to protect against common attacks
  res.set({
    'Strict-Transport-Security': 'max-age=31536000; includeSubDomains', // Forces HTTPS
    'X-Content-Type-Options': 'nosniff',           // Prevents MIME type sniffing attacks
    'X-Frame-Options': 'DENY',                     // Prevents clickjacking attacks
    'X-XSS-Protection': '1; mode=block',           // Enables browser XSS protection
    'Content-Security-Policy': "default-src 'self'" // Restricts resource loading sources
  });
  next();
});
```

# **Flashcards:**

---

What are HTTP Headers and what problem do they solve?;; HTTP Headers are key-value pairs that provide metadata about HTTP requests and responses. They solve the problem of communicating essential information like content type, authentication, caching policies, and connection preferences between clients and servers in a stateless protocol.

What are the four main categories of HTTP headers?;; General headers (apply to both requests and responses), Request headers (client-specific), Response headers (server-specific), and Entity headers (describe the message body content).

What is the purpose of the Content-Type header?;; The Content-Type header specifies the media type (MIME type) of the resource being sent, telling the receiver how to interpret and process the data (e.g., application/json, text/html, image/png).

How do Authorization headers work in HTTP?;; Authorization headers carry authentication credentials from client to server, commonly using formats like "Bearer token" for JWT tokens or "Basic base64credentials" for username/password authentication.

What is the difference between Cache-Control and ETag headers?;; Cache-Control provides caching directives (like max-age, no-cache) that control how long content can be cached, while ETag provides a version identifier that allows conditional requests to check if content has changed.

Which headers are essential for CORS (Cross-Origin Resource Sharing)?;; Key CORS headers include Access-Control-Allow-Origin (specifies allowed origins), Access-Control-Allow-Methods (allowed HTTP methods), Access-Control-Allow-Headers (allowed request headers), and Access-Control-Max-Age (preflight cache duration).