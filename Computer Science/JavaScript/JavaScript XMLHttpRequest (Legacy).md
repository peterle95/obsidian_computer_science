---
tags:
  - learning
language:
  - JavaScript
review-date: 2025-11-25
last-reviewed: 2025-10-08
scheda: done
visit-count: 6
confidence-level: 2
consecutive-correct: 1
last-struggle-date: 2025-09-19
memory: to_finish
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
`XMLHttpRequest` (XHR) solves the fundamental problem of making ==asynchronous HTTP requests from a web browser without requiring a full page reload==. <mark style="background: #D2B3FFA6;">Before XHR, submitting data to a server or fetching new content always necessitated reloading the entire webpage, leading to a clunky and disjointed user experience. XHR revolutionized web development by enabling "Asynchronous JavaScript and XML" (AJAX), allowing web applications to update portions of a page dynamically</mark>.

Its primary application was to facilitate ==dynamic content loading==, form submissions without page refreshes, and building more interactive and responsive web interfaces. It was crucial for JavaScript because it opened up the <mark style="background: #FF5582A6;">possibility of building "single-page applications" (SPAs)</mark> and richer web experiences, moving client-side development closer to desktop application capabilities. In computer science, it represents an early, influential pattern for client-server communication in a web context, pioneering the concept of partial page updates and enabling a new era of web interactivity. Although considered a legacy API now, understanding XHR is essential for comprehending the evolution of web technologies and the underlying mechanisms of more modern fetch APIs.
# **Core Explanation:**
---
`XMLHttpRequest` (XHR) is a <mark style="background: #BBFABBA6;">built-in browser object that provides a way to make HTTP requests to a web server. It allows a web page to request data from the server after the page has loaded, without having to reload the entire page. Despite its name, XHR can work with any type of data, not just XML (e.g., JSON, plain text, HTML).</mark>

Key characteristics and how it works:

- **Asynchronous Nature:** XHR requests are typically ==asynchronous==, meaning the JavaScript code continues to execute while the browser waits for the server's response. This prevents the user interface from freezing.
- **Event-Driven:** XHR operations are driven by events. You attach event listeners to the XHR object to handle different stages of the request (e.g., `onload` for successful completion, `onerror` for network errors, `onprogress` for tracking upload/download progress).
- **Request Lifecycle:** A typical XHR request follows these steps:
    >1. **Creation:** An `XMLHttpRequest` object is instantiated (`new XMLHttpRequest()`).
    >2. **Opening:** The `open()` method is called to specify the HTTP method (e.g., `GET`, `POST`) and the URL.
    >3. **Sending:** The `send()` method initiates the request. For `POST` requests, data can be passed as an argument to `send()`.
    >4. **Handling Response:** Event listeners (most commonly `onload`) process the server's response. The response data is typically available via `xhr.responseText` (for text-based responses) or `xhr.responseXML` (for XML).
    >5. **Status Codes:** The `xhr.status` property provides the HTTP status code (e.g., 200 for OK, 404 for Not Found). The `xhr.readyState` property tracks the state of the request (e.g., `XMLHttpRequest.DONE` for completed).
- **Same-Origin Policy:** ==By default, XHR adheres to the Same-Origin Policy, meaning a request can only be made to the same protocol, port, and host from which the current document was loaded. Cross-Origin Resource Sharing (CORS) mechanisms are needed to enable cross-origin requests.==

While powerful for its time, XHR has a somewhat verbose and callback-heavy API, which can lead to "callback hell" in complex scenarios. This verbosity and the sequential nature of callbacks paved the way for more modern alternatives.
# **Related Concepts:**
---
- **AJAX (Asynchronous JavaScript and XML):** XHR is the core technology that enables AJAX. AJAX is an overarching web development technique for creating asynchronous web applications, while XHR is the specific API used to perform the HTTP requests within the AJAX paradigm.
- **HTTP (Hypertext Transfer Protocol):** XHR is a client-side API for interacting with HTTP servers. It uses HTTP methods (GET, POST, PUT, DELETE, etc.) and processes HTTP status codes. Understanding HTTP fundamentals is crucial for effective XHR usage.
- **JSON (JavaScript Object Notation) and XML (Extensible Markup Language):** While XHR's name suggests XML, JSON became the dominant data format for AJAX responses due to its lightweight nature and native compatibility with JavaScript. XHR can handle both. JSON is easier to parse and work with in JavaScript compared to XML.
- **`fetch()` API:** ==This is the modern, promise-based alternative to `XMLHttpRequest`== for making network requests in web browsers. `fetch()` offers a more streamlined, cleaner API, better error handling, and directly returns Promises, which are easier to manage with `async/await`. While `fetch()` is preferred for new development, XHR still exists for backward compatibility and in some older codebases. `fetch()` is generally more flexible and powerful for handling various response types.
- **Promises and Async/Await:** These JavaScript features are not directly part of XHR but are crucial for managing asynchronous operations in modern JavaScript. They provide a more structured and readable way to handle asynchronous code compared to the traditional callback approach often used with XHR. `fetch()` API inherently uses Promises.
- **CORS (Cross-Origin Resource Sharing):** A security mechanism implemented by browsers that restricts how resources from one origin can be requested from another origin. XHR requests are subject to CORS policies, meaning servers often need to be configured to allow requests from different domains.
# **Examples:**
---
```js
// --- Example 1: Basic GET Request with XMLHttpRequest ---

function getExample() {
    // 1. Create a new XMLHttpRequest object
    const xhr = new XMLHttpRequest();

    // 2. Define what happens when the request loads successfully
    xhr.onload = function() {
        // Check if the request was successful (HTTP status code 200)
        if (xhr.status === 200) {
            console.log("GET Request successful!");
            console.log("Response Text:", xhr.responseText);
            try {
                // If the response is JSON, parse it
                const data = JSON.parse(xhr.responseText);
                console.log("Parsed JSON Data:", data);
            } catch (e) {
                console.warn("Could not parse JSON response:", e);
            }
        } else {
            // Handle HTTP error statuses (e.g., 404 Not Found, 500 Server Error)
            console.error("GET Request failed with status:", xhr.status, xhr.statusText);
        }
    };

    // 3. Define what happens if there's a network error
    xhr.onerror = function() {
        console.error("GET Request failed: Network error occurred.");
    };

    // 4. Define what happens as the request progresses (optional, for larger files)
    xhr.onprogress = function(event) {
        if (event.lengthComputable) {
            console.log(`Received ${event.loaded} of ${event.total} bytes`);
        } else {
            console.log(`Received ${event.loaded} bytes`);
        }
    };

    // 5. Open the request: Specify the HTTP method and URL
    // The third parameter 'true' makes it asynchronous (recommended).
    xhr.open("GET", "https://jsonplaceholder.typicode.com/posts/1", true);

    // 6. Send the request
    xhr.send();
    console.log("GET Request sent..."); // This logs immediately because the request is asynchronous
}

// Call the function to demonstrate GET
getExample();

// --- Example 2: Basic POST Request with XMLHttpRequest ---

function postExample() {
    // 1. Create a new XMLHttpRequest object
    const xhr = new XMLHttpRequest();

    // Data to be sent in the POST request
    const postData = {
        title: 'foo',
        body: 'bar',
        userId: 1
    };

    // 2. Define what happens when the request loads successfully
    xhr.onload = function() {
        if (xhr.status >= 200 && xhr.status < 300) { // Check for 2xx success status codes
            console.log("POST Request successful!");
            console.log("Response Text:", xhr.responseText);
            try {
                const responseData = JSON.parse(xhr.responseText);
                console.log("Parsed JSON Response:", responseData);
            } catch (e) {
                console.warn("Could not parse JSON response for POST:", e);
            }
        } else {
            console.error("POST Request failed with status:", xhr.status, xhr.statusText);
        }
    };

    // 3. Define what happens if there's a network error
    xhr.onerror = function() {
        console.error("POST Request failed: Network error occurred.");
    };

    // 4. Open the request: Method is POST, URL is the endpoint
    xhr.open("POST", "https://jsonplaceholder.typicode.com/posts", true);

    // 5. Set the Content-Type header for JSON data
    // This tells the server that the body of the request is JSON
    xhr.setRequestHeader("Content-Type", "application/json;charset=UTF-8");

    // 6. Send the request with the JSON stringified data
    xhr.send(JSON.stringify(postData));
    console.log("POST Request sent...");
}

// Call the function to demonstrate POST after a short delay
// to avoid log clutter from the GET example.
setTimeout(postExample, 2000);

// --- Example 3: Monitoring ReadyState Changes (more detailed, less common now) ---

function readyStateExample() {
    const xhr = new XMLHttpRequest();
    const url = "https://jsonplaceholder.typicode.com/todos/1";

    xhr.onreadystatechange = function() {
        // xhr.readyState has different states:
        // 0: UNSENT (client has been created. open() not called yet.)
        // 1: OPENED (open() has been called.)
        // 2: HEADERS_RECEIVED (send() has been called, and headers and status are available.)
        // 3: LOADING (Downloading; responseText holds partial data.)
        // 4: DONE (The operation is complete.)
        console.log(`ReadyState: ${xhr.readyState}, Status: ${xhr.status}`);

        if (xhr.readyState === XMLHttpRequest.DONE) { // Equivalent to xhr.readyState === 4
            if (xhr.status === 200) {
                console.log("DONE state, Success!");
                console.log("Response (ReadyState):", xhr.responseText);
            } else {
                console.error("DONE state, Error! Status:", xhr.status);
            }
        }
    };

    xhr.open("GET", url, true);
    xhr.send();
    console.log("ReadyState tracking request sent...");
}

// Call the function to demonstrate readyState changes
setTimeout(readyStateExample, 4000); // Give previous examples time to log
```
# **Flashcards:**
---
What problem does `XMLHttpRequest` solve?;; It enables asynchronous HTTP requests from the browser, allowing dynamic page updates without full reloads (AJAX). 

What are the key steps in making an `XMLHttpRequest`?;; Create XHR object, open (method, URL, async), set event handlers (onload, onerror), set request headers (if needed), send (data if POST/PUT). 

Is `XMLHttpRequest` related to JSX?;; No, `XMLHttpRequest` is for network requests, while JSX is a JavaScript syntax extension for describing UI structure.