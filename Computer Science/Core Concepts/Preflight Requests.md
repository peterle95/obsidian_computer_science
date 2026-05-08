---
memory: to_finish
tags:
  - to_learn
language:
  - Core Concepts
review-date: 2025-11-20
last-reviewed: 2025-10-21
scheda: done
visit-count: 4
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-09-24
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
# Purpose/Why:
---

Preflight requests provide a <mark style="background: #FF5582A6;">security mechanism within the CORS</mark> (Cross-Origin Resource Sharing) framework that <mark style="background: #FF5582A6;">helps protect servers from potentially harmful cross-origin HTTP requests</mark>. By <mark style="background: #FF5582A6;">requiring browsers to "check" with the server before sending certain types of requests, preflight requests give servers the opportunity to verify whether the actual request should be allowed</mark>, thereby preventing unauthorized or potentially dangerous operations. This is crucial for maintaining security when relaxing the [[Same-Origin Policy (SOP)]] to enable controlled cross-origin data sharing.

# Core Explanation:
---

A preflight request is a<mark style="background: #ABF7F7A6;"> preliminary HTTP request that the browser sends before the actual request when making certain cross-origin requests</mark>. It uses the <mark style="background: #ABF7F7A6;">HTTP OPTIONS method to ask the server if the actual request (with its specific method, headers, etc.) is acceptable.</mark>

**Key Characteristics:**
1. **Automatic and Transparent**: Browsers automatically send preflight requests without developer intervention.
2. **OPTIONS Method**: ==Always uses the HTTP OPTIONS method.==
3. **Special Headers**:
  >- `Origin`: Indicates where the request originates from
   >- `Access-Control-Request-Method`: Specifies the method of the actual request
  > - `Access-Control-Request-Headers`: Lists any custom headers the actual request will include

**When Preflight Occurs:**
==Not all cross-origin requests trigger a preflight. A request requires preflight when:==
- It uses methods other than GET, POST, or HEAD
- It includes headers beyond the "simple headers" (Accept, Accept-Language, Content-Language, Content-Type)
- The Content-Type header has values other than application/x-www-form-urlencoded, multipart/form-data, or text/plain
- It uses ReadableStream objects
- It includes event listeners on XMLHttpRequestUpload

**Server Response:**
<mark style="background: #ADCCFFA6;">The server must respond with appropriate CORS headers:</mark>
- `Access-Control-Allow-Origin`: Which origins are allowed
- `Access-Control-Allow-Methods`: Which methods are permitted
- `Access-Control-Allow-Headers`: Which headers are accepted
- `Access-Control-Max-Age`: How long the preflight results can be cached

Only if the preflight response indicates permission will the browser proceed with the actual request.

# Related Concepts:
---

- **[[CORS (Cross-Origin Resource Sharing)]]**: The broader framework that includes preflight requests. CORS enables controlled cross-origin requests through HTTP headers, with preflight being one mechanism within this system.

- **Simple Requests**: Cross-origin requests that don't trigger a preflight. They must use only certain methods (GET, POST, HEAD) and headers. Understanding what makes a request "simple" helps determine when preflights occur.

- **Same-Origin Policy**: The security policy that CORS and preflight requests modify. While Same-Origin Policy restricts cross-origin access, CORS with preflight provides a secure way to relax these restrictions.

- **HTTP OPTIONS Method**: The HTTP method used for preflight requests. While OPTIONS can be used for other purposes (like discovering server capabilities), in CORS it's specifically used for preflight checks.

- **CORS Headers**: The HTTP headers that control cross-origin access. Preflight requests rely on specific CORS headers to determine if the actual request is permitted.

- **Credentialed Requests**: Cross-origin requests that include credentials (cookies, HTTP authentication). These have special preflight considerations and require specific CORS headers.

- **CSRF (Cross-Site Request Forgery)**: An attack that CORS and preflight requests help mitigate by preventing unauthorized cross-origin requests from executing without proper server validation.

# Examples:
---


```javascript
// CLIENT-SIDE EXAMPLE: Request that will trigger a preflight

// This fetch request will trigger a preflight because:
// 1. It uses the PUT method (not GET, POST, or HEAD)
// 2. It includes a custom header (X-Custom-Header)
fetch('https://api.example.com/update-data', {
  method: 'PUT', // Non-simple method that triggers preflight
  headers: {
    'Content-Type': 'application/json', // This content-type is allowed but combined with PUT method will trigger preflight
    'X-Custom-Header': 'value' // Custom header triggers preflight
  },
  body: JSON.stringify({ key: 'value' })
})
  .then(response => response.json())
  .then(data => console.log('Success:', data))
  .catch(error => console.error('Error:', error));

// What happens behind the scenes:
// 1. Browser sends a preflight OPTIONS request:
//    OPTIONS /update-data HTTP/1.1
//    Host: api.example.com
//    Origin: https://your-site.com
//    Access-Control-Request-Method: PUT
//    Access-Control-Request-Headers: Content-Type, X-Custom-Header
//
// 2. Server must respond with appropriate headers:
//    HTTP/1.1 200 OK
//    Access-Control-Allow-Origin: https://your-site.com
//    Access-Control-Allow-Methods: PUT, GET, POST
//    Access-Control-Allow-Headers: Content-Type, X-Custom-Header
//    Access-Control-Max-Age: 86400
//
// 3. Only if the preflight succeeds, the browser sends the actual PUT request


// CLIENT-SIDE EXAMPLE: Request that won't trigger a preflight

// This fetch request will NOT trigger a preflight because:
// 1. It uses GET method (a simple method)
// 2. It doesn't include any custom headers
fetch('https://api.example.com/data')
  .then(response => response.json())
  .then(data => console.log('Data received without preflight:', data))
  .catch(error => console.error('Error:', error));


// SERVER-SIDE EXAMPLE (Node.js with Express)
const express = require('express');
const app = express();

// Middleware to handle CORS preflight requests
app.options('/update-data', (req, res) => {
  // Check origin of the request
  const origin = req.headers.origin;
  
  // Check if this origin is allowed
  if (origin === 'https://allowed-site.com') {
    // Respond to preflight request with appropriate CORS headers
    res.header('Access-Control-Allow-Origin', origin);
    res.header('Access-Control-Allow-Methods', 'PUT, POST, GET, DELETE, OPTIONS');
    res.header('Access-Control-Allow-Headers', 'Content-Type, X-Custom-Header');
    res.header('Access-Control-Max-Age', '86400'); // Cache preflight results for 24 hours
    res.sendStatus(204); // No content needed for OPTIONS response
  } else {
    // Reject preflight from unauthorized origins
    res.sendStatus(403);
  }
});

// The actual endpoint that will be accessed if preflight succeeds
app.put('/update-data', (req, res) => {
  // Set CORS headers for the actual response too
  res.header('Access-Control-Allow-Origin', 'https://allowed-site.com');
  
  // Process the request and send response
  res.json({ status: 'Data updated successfully' });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

# Flashcards:

---

What is a preflight request in CORS?;; A preliminary HTTP OPTIONS request that the browser automatically sends before certain cross-origin requests to check if the server allows the actual request with its specific method and headers.

When does a cross-origin request trigger a preflight request?;; When it uses methods other than GET, POST, or HEAD; includes non-simple headers; uses Content-Type values other than application/x-www-form-urlencoded, multipart/form-data, or text/plain; or uses ReadableStream objects.

What HTTP method is used for preflight requests?;; The OPTIONS method.

What are the key request headers in a preflight request?;; Origin, Access-Control-Request-Method, and Access-Control-Request-Headers.

What response headers must a server include to approve a preflight request?;; Access-Control-Allow-Origin, Access-Control-Allow-Methods, and Access-Control-Allow-Headers. Optionally Access-Control-Max-Age to enable caching.

What is the purpose of the Access-Control-Max-Age header in a preflight response?;; It specifies how long (in seconds) the browser can cache the preflight results, avoiding repeated preflight requests for the same type of request.