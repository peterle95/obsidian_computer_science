---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-09-01
scheda: done
visit-count: 5
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-08-10
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

CORS solves the fundamental security problem of controlling ==cross-origin requests in web browsers while enabling legitimate cross-domain communication==. The ==[[Same-Origin Policy (SOP)]] blocks web pages from making requests to different domains, ports, or protocols to prevent malicious scripts from accessing sensitive data. However, modern web applications often need to communicate with APIs on different domains. CORS provides a secure mechanism for servers to explicitly allow specific cross-origin requests by using [[HTTP Headers]].== It's crucial for web security because it ==prevents unauthorized access to resources while enabling legitimate use cases like APIs, CDNs, and microservices.== Without CORS, either all cross-origin requests would be blocked (breaking modern web apps) or allowed (creating massive security vulnerabilities).

# **Core Explanation:**
---

CORS (Cross-Origin Resource Sharing) is a ==browser security mechanism that uses HTTP headers== to control <mark style="background: #FFF3A3A6;">which web pages can access resources from different origins (domain, protocol, or port</mark>). 

>When a web page makes a cross-origin request, the browser <mark style="background: #BBFABBA6;">first checks if the request is "simple" </mark><mark style="background: #BBFABBA6;"></mark>(GET, POST with specific content types) or requires a preflight check. For complex requests, the browser sends an OPTIONS request (preflight) to ask the server if the actual request is allowed. The server responds with CORS headers like `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, and `Access-Control-Allow-Headers`. If the server permits the request, the browser proceeds; otherwise, it blocks the request and throws a CORS error.

<mark style="background: #FF5582A6;">CORS is entirely browser-enforced</mark> - servers receive all requests regardless, but browsers block the response from reaching the JavaScript code if CORS headers don't permit it.

**CORS (Cross-Origin Resource Sharing)** is a browser security mechanism that restricts web pages from making requests to a different domain than the one that served the web page. This is a fundamental security feature called the **same-origin policy**, designed to prevent malicious websites from reading data from other websites.

In your case, you encountered a CORS error because your frontend application was running on `http://127.0.0.1:3000` (port 3000) and was attempting to make an API request to your backend application running on `http://127.0.0.1:8001` (port 8001). Even though both are on `127.0.0.1`, ==different **ports** are considered different "origins" by the browser's same-origin policy.==

When a browser makes a "cross-origin" request (a request to a different origin than the one that served the page), it first sends a **preflight request** (an `OPTIONS` HTTP method) to the server. This preflight request checks if the server is willing to accept the actual request. If the server doesn't respond with the appropriate **CORS headers**, the browser blocks the actual request, resulting in a CORS error
# **Related Concepts:**
---

- **Same-Origin Policy (SOP)**: The fundamental browser security model that CORS extends - blocks requests between different origins by default. 
- **[[Preflight Requests]]**: OPTIONS requests sent by browsers to check if complex cross-origin requests are allowed. 
- **JSONP**: An older workaround for cross-origin requests using script tags, largely superseded by CORS. 
- **CSP (Content Security Policy)**: Another security mechanism that controls resource loading but focuses on preventing XSS attacks. 
- **HTTP Headers**: CORS relies entirely on HTTP headers for communication between browser and server. 
- **XMLHttpRequest/Fetch API**: The JavaScript APIs that trigger CORS checks when making cross-origin requests. 
- **CSRF (Cross-Site Request Forgery)**: A security vulnerability that CORS helps prevent by controlling cross-origin access.

# **Examples:**
---

```javascript
// CLIENT-SIDE EXAMPLES

// Simple cross-origin request (might work without preflight)
fetch('https://api.example.com/data', {
    method: 'GET', // Simple method
    headers: {
        'Content-Type': 'application/json' // Simple content type
    }
})
.then(response => {
    if (!response.ok) {
        throw new Error('Network response was not ok');
    }
    return response.json();
})
.then(data => console.log(data))
.catch(error => {
    // This will catch CORS errors among others
    console.error('CORS or network error:', error);
});

// Complex cross-origin request (requires preflight)
fetch('https://api.example.com/secure-data', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer token123', // Custom header triggers preflight
        'X-Custom-Header': 'value' // Any custom header triggers preflight
    },
    body: JSON.stringify({
        userId: 123,
        action: 'update'
    })
})
.then(response => response.json())
.then(data => console.log(data))
.catch(error => {
    // CORS errors will be caught here
    console.error('Request failed:', error);
});

// XMLHttpRequest example with CORS
const xhr = new XMLHttpRequest();
xhr.open('GET', 'https://api.different-domain.com/data');

// Set up event handlers
xhr.onload = function() {
    if (xhr.status === 200) {
        console.log('Success:', xhr.responseText);
    } else {
        console.error('HTTP error:', xhr.status);
    }
};

xhr.onerror = function() {
    // This fires for network errors, including CORS failures
    console.error('Network error or CORS blocked');
};

// Adding custom headers will trigger preflight
xhr.setRequestHeader('X-API-Key', 'secret-key');
xhr.send();
````

```javascript
// SERVER-SIDE EXAMPLES (Node.js/Express)

const express = require('express');
const app = express();

// Basic CORS middleware - allows all origins (NOT recommended for production)
app.use((req, res, next) => {
    res.header('Access-Control-Allow-Origin', '*'); // Allow all origins
    res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
    res.header('Access-Control-Allow-Headers', 'Content-Type, Authorization');
    next();
});

// More restrictive CORS - only allow specific origin
app.use((req, res, next) => {
    const allowedOrigin = 'https://myapp.com';
    const origin = req.headers.origin;
    
    if (origin === allowedOrigin) {
        res.header('Access-Control-Allow-Origin', origin); // Echo back the allowed origin
    }
    
    res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
    res.header('Access-Control-Allow-Headers', 'Content-Type, Authorization');
    
    // Handle preflight requests
    if (req.method === 'OPTIONS') {
        res.header('Access-Control-Max-Age', '86400'); // Cache preflight for 24 hours
        return res.status(200).end();
    }
    
    next();
});

// Dynamic CORS based on request
app.use((req, res, next) => {
    const allowedOrigins = [
        'https://myapp.com',
        'https://admin.myapp.com',
        'http://localhost:3000' // For development
    ];
    
    const origin = req.headers.origin;
    
    if (allowedOrigins.includes(origin)) {
        res.header('Access-Control-Allow-Origin', origin);
        res.header('Access-Control-Allow-Credentials', 'true'); // Allow cookies
    }
    
    res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
    res.header('Access-Control-Allow-Headers', 'Content-Type, Authorization, X-Requested-With');
    
    if (req.method === 'OPTIONS') {
        return res.status(200).end();
    }
    
    next();
});

// API endpoint that returns data
app.get('/api/data', (req, res) => {
    // The CORS headers set above will be included in this response
    res.json({
        message: 'This data can be accessed cross-origin',
        timestamp: new Date().toISOString()
    });
});

// API endpoint that requires authentication
app.post('/api/secure', (req, res) => {
    // Check if the request has proper authorization
    const authHeader = req.headers.authorization;
    
    if (!authHeader) {
        return res.status(401).json({ error: 'No authorization header' });
    }
    
    // CORS headers are already set by middleware above
    res.json({
        message: 'Secure data accessed successfully',
        user: 'authenticated-user'
    });
});

app.listen(3000, () => {
    console.log('Server running on port 3000');
});
```

```javascript
// USING CORS LIBRARY (Express.js)
const express = require('express');
const cors = require('cors');
const app = express();

// Simple CORS setup
app.use(cors());

// Advanced CORS configuration
const corsOptions = {
    origin: function (origin, callback) {
        // Allow requests with no origin (mobile apps, server-to-server)
        if (!origin) return callback(null, true);
        
        const allowedOrigins = [
            'https://myapp.com',
            'https://admin.myapp.com'
        ];
        
        if (allowedOrigins.includes(origin)) {
            callback(null, true); // Allow this origin
        } else {
            callback(new Error('Not allowed by CORS')); // Block this origin
        }
    },
    credentials: true, // Allow cookies and authentication headers
    optionsSuccessStatus: 200, // Some legacy browsers choke on 204
    methods: ['GET', 'POST', 'PUT', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization', 'X-Requested-With'],
    exposedHeaders: ['X-Total-Count'], // Headers that client can access
    maxAge: 86400 // How long browsers can cache preflight results
};

app.use(cors(corsOptions));

// Route-specific CORS
app.get('/public-api', cors(), (req, res) => {
    // This route allows all origins
    res.json({ message: 'Public data' });
});

app.get('/admin-api', cors({
    origin: 'https://admin.myapp.com'
}), (req, res) => {
    // This route only allows admin domain
    res.json({ message: 'Admin data' });
});
```

# **Flashcards:** 
---

What is CORS and why is it needed?;; CORS (Cross-Origin Resource Sharing) is a browser security mechanism that uses HTTP headers to control cross-origin requests. It's needed because the Same-Origin Policy blocks all cross-origin requests by default, but modern web apps need to communicate with APIs on different domains.

What triggers a CORS preflight request?;; A preflight request is triggered by "complex" requests: non-simple HTTP methods (PUT, DELETE, etc.), custom headers beyond simple ones, or content types other than application/x-www-form-urlencoded, multipart/form-data, or text/plain.

What are the key CORS headers a server must send?;; `Access-Control-Allow-Origin` (specifies allowed origins), `Access-Control-Allow-Methods` (allowed HTTP methods), `Access-Control-Allow-Headers` (allowed request headers), and optionally `Access-Control-Allow-Credentials` (allows cookies/auth).

What's the difference between simple and complex CORS requests?;; Simple requests (GET, POST with basic content types and no custom headers) are sent directly. Complex requests trigger a preflight OPTIONS request first to check if the actual request is allowed by the server.

Where are CORS checks enforced and why can't you bypass them?;; CORS is enforced entirely by the browser, not the server. Servers receive all requests regardless of CORS, but browsers block the response from reaching JavaScript if CORS headers don't permit it. You can't bypass it from client-side code because it's a browser security feature.

How do you handle CORS for requests with credentials (cookies)?;; Set `Access-Control-Allow-Credentials: true` on the server and `credentials: 'include'` in the fetch request. The `Access-Control-Allow-Origin` cannot be `*` when credentials are allowed - it must specify exact origins.