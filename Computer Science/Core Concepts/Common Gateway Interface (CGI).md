---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-09-17
scheda: done
visit-count: 5
confidence-level: 2.5
consecutive-correct: 3
last-struggle-date: 2025-08-16
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

CGI solves the fundamental problem of ==enabling web servers to execute external programs and generate dynamic content in response to HTTP requests==. ==Before CGI, web servers could only serve static files==. CGI provides a standardized interface that allows web servers to communicate with external applications (written in any programming language) to process user input, interact with databases, and <mark style="background: #FF5582A6;">generate dynamic HTML responses</mark>. This was crucial for the early development of interactive web applications and e-commerce sites, making the web more than just a collection of static documents.

# **Core Explanation:**
---

Common Gateway Interface (CGI) is a standard protocol that defines how web servers communicate with external programs to generate dynamic web content. <mark style="background: #BBFABBA6;">When a web server receives a request for a CGI script, it launches the script as a separate process, passes request data through environment variables and standard input, and captures the script's output to send back to the client.</mark>

**Key Characteristics:**
- **Language Independent**: CGI scripts can be written in many programming language (Perl, Python, C, shell scripts, etc.)
- **Process-based**: ==Each request spawns a new process, providing isolation but with performance overhead==
- **Stateless**: Each request is independent; no memory is shared between requests
- **Standard Interface**: Uses environment variables (QUERY_STRING, REQUEST_METHOD, etc.) and standard I/O for communication

**How it works:**
1. <mark style="background: #FF5582A6;">Client sends HTTP request to web server</mark>
2. <mark style="background: #ADCCFFA6;">Server identifies the request as targeting a CGI script</mark>
3. <mark style="background: #ADCCFFA6;">Server sets environment variables with request information</mark>
4. <mark style="background: #ADCCFFA6;">Server launches the CGI script as a new process</mark>
5. <mark style="background: #BBFABBA6;">Script processes input, performs operations </mark>(database queries, calculations, etc.)
6. <mark style="background: #BBFABBA6;">Script outputs HTTP headers and content to standard output</mark>
7. <mark style="background: #D2B3FFA6;">Server captures the output and sends it back to the client</mark>
8. Process terminates

# **Related Concepts:**
---

**FastCGI**: An improvement over CGI that keeps processes alive between requests, reducing the overhead of process creation. Scripts run as persistent processes that handle multiple requests.

**Server-Side Includes (SSI)**: A simpler server-side technology for including dynamic content in static HTML pages, but with limited functionality compared to CGI.

**Application Servers**: Modern alternatives like Apache Tomcat, [[Node.js - Server-Side JavaScript]], or Python WSGI servers that provide more efficient ways to run server-side applications with better performance and resource management.

**[[Environment Variables (CGI)]]**: CGI heavily relies on environment variables (REQUEST_METHOD, QUERY_STRING, CONTENT_TYPE) to pass request information from the server to the script.

**HTTP Protocol**: CGI scripts must understand HTTP to generate proper responses with correct headers (Content-Type, Status codes, etc.).

**Web Server Modules**: Modern alternatives like Apache modules (mod_php, mod_python) or Nginx modules that embed interpreters directly into the web server process.

# **Examples:**
---

```python
#!/usr/bin/env python3
# Shebang line tells the server which interpreter to use
# This must be the first line in a CGI script

import cgi
import os
import sys

# CGI scripts must output HTTP headers first
# Content-Type header is mandatory - tells browser what type of content follows
print("Content-Type: text/html\n")

# HTML content begins here
print("<html><head><title>CGI Example</title></head><body>")
print("<h1>CGI Environment Information</h1>")

# Access environment variables set by the web server
# These contain information about the HTTP request
request_method = os.environ.get('REQUEST_METHOD', 'Unknown')
query_string = os.environ.get('QUERY_STRING', '')
remote_addr = os.environ.get('REMOTE_ADDR', 'Unknown')

print(f"<p><strong>Request Method:</strong> {request_method}</p>")
print(f"<p><strong>Query String:</strong> {query_string}</p>")
print(f"<p><strong>Client IP:</strong> {remote_addr}</p>")

# Process form data if this is a POST request
if request_method == 'POST':
    # Create FieldStorage object to parse form data
    form = cgi.FieldStorage()
    
    # Check if 'name' field was submitted
    if 'name' in form:
        name = form['name'].value
        print(f"<p><strong>Hello, {name}!</strong></p>")

# Display a simple form for user input
print("""
<hr>
<h2>Submit Your Name</h2>
<form method="post" action="">
    <label for="name">Name:</label>
    <input type="text" id="name" name="name" required>
    <input type="submit" value="Submit">
</form>
""")

print("</body></html>")
````

```bash
#!/bin/bash
# Simple shell script CGI example
# Shows how CGI can be implemented in any language

# Output HTTP headers - Content-Type is required
echo "Content-Type: text/html"
echo ""  # Empty line separates headers from content

# Generate HTML response
echo "<html><head><title>System Info CGI</title></head><body>"
echo "<h1>Server System Information</h1>"

# Use environment variables provided by the web server
echo "<p><strong>Server Name:</strong> $SERVER_NAME</p>"
echo "<p><strong>Server Port:</strong> $SERVER_PORT</p>"
echo "<p><strong>Request URI:</strong> $REQUEST_URI</p>"

# Execute system commands and include output in response
echo "<h2>Current Date and Time:</h2>"
echo "<pre>$(date)</pre>"

echo "<h2>Server Uptime:</h2>"
echo "<pre>$(uptime)</pre>"

echo "</body></html>"
```
# **Flashcards:**
---

What is the primary purpose of CGI (Common Gateway Interface)?;; CGI enables web servers to execute external programs and generate dynamic content in response to HTTP requests, solving the limitation of serving only static files.

How does CGI pass request information from the web server to the script?;; CGI uses environment variables (like REQUEST_METHOD, QUERY_STRING, REMOTE_ADDR) and standard input to pass HTTP request data to the external program.

What are the main performance limitations of traditional CGI?;; CGI creates a new process for each request, which introduces significant overhead in process creation and termination, making it slower than modern alternatives like FastCGI or application servers.

What must every CGI script output before sending content to the client?;; Every CGI script must output HTTP headers first, with Content-Type being mandatory, followed by a blank line before sending the actual content.

Name three programming languages that can be used to write CGI scripts and why this flexibility exists.;; Any programming language can be used (Python, Perl, C, Shell scripts, etc.) because CGI is a standard protocol that communicates through environment variables and standard I/O, not language-specific APIs.

What is the difference between CGI and FastCGI?;; CGI creates a new process for each request and terminates it afterward, while FastCGI keeps processes alive between requests, significantly reducing the overhead of process creation and improving performance.