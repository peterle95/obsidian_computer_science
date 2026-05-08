---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-10-17
scheda: done
visit-count: 6
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-09-09
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

CGI environment variables solve the fundamental problem of ==standardized communication between web servers and external scripts==. They provide a ==consistent, language-independent way to pass HTTP request information (headers, client data, server configuration) from the web server to CGI scripts==. This eliminates the need for proprietary APIs or custom communication protocols, allowing any programming language to process web requests. They're crucial because they ==maintain the stateless nature of HTTP while providing scripts with all necessary context about the request, client, and server environment.==

# **Core Explanation:**
---

CGI environment variables are a <mark style="background: #BBFABBA6;">specific set of predefined variables that web servers automatically populate with HTTP request information before executing CGI scripts</mark>. These variables follow the CGI specification (RFC 3875) and provide a standardized interface for web communication.

**Key Characteristics:**
- **Standardized Names**: Follow specific naming conventions (SERVER_NAME, REQUEST_METHOD, etc.)
- **Automatically Set**: Web server populates them <mark style="background: #FF5582A6;">based on incoming HTTP requests</mark>
- **Read-Only**: Scripts can read but typically shouldn't modify these values
- **Request-Specific**: Each CGI process gets variables specific to that HTTP request
- **Case-Sensitive**: Variable names are typically uppercase by convention

**Main Categories:**
1. **Request Variables**: Information about the HTTP request (METHOD, URI, QUERY_STRING)
2. **Server Variables**: Information about the web server (SERVER_NAME, SERVER_PORT)
3. **Client Variables**: Information about the client (REMOTE_ADDR, USER_AGENT)
4. **Protocol Variables**: HTTP-specific information (HTTP_*, CONTENT_TYPE, CONTENT_LENGTH)

**How they differ from regular Linux environment variables:**
- **Scope**: CGI variables exist only during script execution, while user environment variables persist across shell sessions
- **Source**: CGI variables are set by the web server based on HTTP requests, while user variables are set by shell, login scripts, or manual export
- **Purpose**: CGI variables carry web request data, while user variables configure system behavior and user preferences
- **Naming**: CGI variables follow web-specific conventions, while user variables can be arbitrary
- **Inheritance**: User environment variables inherit from parent processes; CGI variables are freshly created for each request

# **Related Concepts:**
---

**Linux Environment Variables**: System-wide or user-specific variables (PATH, HOME, USER) that configure system behavior and are inherited by child processes. CGI variables are a specialized subset created specifically for web requests.

**[[HTTP Headers]]**: The original source of many CGI environment variables. Headers like "User-Agent" become "HTTP_USER_AGENT" in CGI, with specific transformation rules.

**Process Environment**: In Linux, each process has an environment block containing key-value pairs. CGI scripts inherit this plus the additional CGI-specific variables.

**Shell Variables vs Environment Variables**: Shell variables exist only in the current shell, while environment variables are passed to child processes. CGI variables behave like environment variables for the script's duration.

**FastCGI Environment**: Uses similar variable concepts but may persist them across requests for performance, unlike traditional CGI which recreates them each time.

**WSGI Environment**: Python's Web Server Gateway Interface uses a similar concept but passes information through a Python dictionary rather than environment variables.

# **Examples:**
---

```python
#!/usr/bin/env python3
# CGI Environment Variables Demo Script

import os
import sys

# Output required HTTP headers
print("Content-Type: text/html\n")
print("<html><head><title>CGI Environment Variables</title></head><body>")
print("<h1>CGI Environment Variables vs Regular Environment Variables</h1>")

# Standard CGI Variables - These are set by the web server
# and contain information about the HTTP request
print("<h2>Standard CGI Variables</h2>")
print("<table border='1' style='border-collapse: collapse;'>")
print("<tr><th>Variable</th><th>Value</th><th>Description</th></tr>")

# REQUEST_METHOD: HTTP method used (GET, POST, PUT, etc.)
request_method = os.environ.get('REQUEST_METHOD', 'Not Set')
print(f"<tr><td>REQUEST_METHOD</td><td>{request_method}</td><td>HTTP method used for this request</td></tr>")

# QUERY_STRING: Everything after the ? in the URL
query_string = os.environ.get('QUERY_STRING', 'Not Set')
print(f"<tr><td>QUERY_STRING</td><td>{query_string}</td><td>URL parameters passed in GET request</td></tr>")

# SERVER_NAME: The server's hostname or IP address
server_name = os.environ.get('SERVER_NAME', 'Not Set')
print(f"<tr><td>SERVER_NAME</td><td>{server_name}</td><td>Web server's hostname</td></tr>")

# SERVER_PORT: Port number the server is listening on
server_port = os.environ.get('SERVER_PORT', 'Not Set')
print(f"<tr><td>SERVER_PORT</td><td>{server_port}</td><td>Port number for this request</td></tr>")

# REMOTE_ADDR: Client's IP address
remote_addr = os.environ.get('REMOTE_ADDR', 'Not Set')
print(f"<tr><td>REMOTE_ADDR</td><td>{remote_addr}</td><td>IP address of the client</td></tr>")

# CONTENT_TYPE: MIME type of request body (important for POST requests)
content_type = os.environ.get('CONTENT_TYPE', 'Not Set')
print(f"<tr><td>CONTENT_TYPE</td><td>{content_type}</td><td>MIME type of request body</td></tr>")

# CONTENT_LENGTH: Size of request body in bytes
content_length = os.environ.get('CONTENT_LENGTH', 'Not Set')
print(f"<tr><td>CONTENT_LENGTH</td><td>{content_length}</td><td>Size of request body</td></tr>")

print("</table>")

# HTTP Headers become environment variables with HTTP_ prefix
# and header names converted to uppercase with dashes becoming underscores
print("<h2>HTTP Headers as Environment Variables</h2>")
print("<table border='1' style='border-collapse: collapse;'>")
print("<tr><th>Variable</th><th>Value</th><th>Original Header</th></tr>")

# USER_AGENT header becomes HTTP_USER_AGENT
user_agent = os.environ.get('HTTP_USER_AGENT', 'Not Set')
print(f"<tr><td>HTTP_USER_AGENT</td><td>{user_agent}</td><td>User-Agent</td></tr>")

# ACCEPT header becomes HTTP_ACCEPT
accept = os.environ.get('HTTP_ACCEPT', 'Not Set')
print(f"<tr><td>HTTP_ACCEPT</td><td>{accept}</td><td>Accept</td></tr>")

# HOST header becomes HTTP_HOST
host = os.environ.get('HTTP_HOST', 'Not Set')
print(f"<tr><td>HTTP_HOST</td><td>{host}</td><td>Host</td></tr>")

print("</table>")

# Regular Linux Environment Variables - These exist for all processes
# and are not specific to web requests
print("<h2>Regular Linux Environment Variables (Non-CGI)</h2>")
print("<table border='1' style='border-collapse: collapse;'>")
print("<tr><th>Variable</th><th>Value</th><th>Purpose</th></tr>")

# PATH: Directories to search for executable files
path = os.environ.get('PATH', 'Not Set')
print(f"<tr><td>PATH</td><td>{path[:100]}...</td><td>Executable search path</td></tr>")

# HOME: User's home directory (may not be set in CGI context)
home = os.environ.get('HOME', 'Not Set')
print(f"<tr><td>HOME</td><td>{home}</td><td>User home directory</td></tr>")

# USER: Current username (may not be set in CGI context)
user = os.environ.get('USER', 'Not Set')
print(f"<tr><td>USER</td><td>{user}</td><td>Current user name</td></tr>")

# SHELL: User's default shell
shell = os.environ.get('SHELL', 'Not Set')
print(f"<tr><td>SHELL</td><td>{shell}</td><td>User's default shell</td></tr>")

print("</table>")

print("</body></html>")
```

```bash
#!/bin/bash
# Bash script demonstrating CGI environment variables vs regular environment

echo "Content-Type: text/html"
echo ""  # Required blank line

echo "<html><head><title>Environment Variable Comparison</title></head><body>"
echo "<h1>CGI vs Regular Environment Variables</h1>"

echo "<h2>Key Differences Demonstration</h2>"
echo "<pre>"

# Show that CGI variables are web-specific and temporary
echo "=== CGI-Specific Variables (only exist during web request) ==="
echo "REQUEST_METHOD: $REQUEST_METHOD"
echo "SERVER_NAME: $SERVER_NAME" 
echo "QUERY_STRING: $QUERY_STRING"
echo "REMOTE_ADDR: $REMOTE_ADDR"

echo ""
echo "=== Regular Environment Variables (inherited from system) ==="
echo "PATH: ${PATH:0:60}..."  # Truncate long PATH for display
echo "HOME: $HOME"
echo "USER: $USER"
echo "SHELL: $SHELL"

echo ""
echo "=== Process Information ==="
# Show process ID - each CGI request gets a new process
echo "Current Process ID: $$"
echo "Parent Process ID: $PPID"

# Demonstrate that we can access both types simultaneously
echo ""
echo "=== Accessing Both Types in Same Script ==="
if [ -n "$REQUEST_METHOD" ]; then
    echo "This script was called via HTTP $REQUEST_METHOD"
else
    echo "This script was not called via web server"
fi

if [ -n "$HOME" ]; then
    echo "Running with HOME directory: $HOME"
else
    echo "No HOME directory set (common in CGI)"
fi

echo "</pre>"
echo "</body></html>"
```

```c
// C program showing CGI environment variable access
// Compile with: gcc -o cgi_env.cgi cgi_env.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {
    // Output HTTP headers first
    printf("Content-Type: text/html\n\n");
    
    printf("<html><head><title>CGI Environment in C</title></head><body>\n");
    printf("<h1>CGI Environment Variables in C</h1>\n");
    
    // Access CGI-specific environment variables using getenv()
    // getenv() searches the environment for the named variable
    char *request_method = getenv("REQUEST_METHOD");
    char *query_string = getenv("QUERY_STRING");
    char *server_name = getenv("SERVER_NAME");
    char *remote_addr = getenv("REMOTE_ADDR");
    
    printf("<h2>CGI Variables</h2>\n");
    printf("<p><strong>REQUEST_METHOD:</strong> %s</p>\n", 
           request_method ? request_method : "Not Set");
    printf("<p><strong>QUERY_STRING:</strong> %s</p>\n", 
           query_string ? query_string : "Not Set");
    printf("<p><strong>SERVER_NAME:</strong> %s</p>\n", 
           server_name ? server_name : "Not Set");
    printf("<p><strong>REMOTE_ADDR:</strong> %s</p>\n", 
           remote_addr ? remote_addr : "Not Set");
    
    // Access regular environment variables
    char *path = getenv("PATH");
    char *home = getenv("HOME");
    char *user = getenv("USER");
    
    printf("<h2>Regular Environment Variables</h2>\n");
    printf("<p><strong>PATH:</strong> %.100s%s</p>\n", 
           path ? path : "Not Set", path && strlen(path) > 100 ? "..." : "");
    printf("<p><strong>HOME:</strong> %s</p>\n", 
           home ? home : "Not Set");
    printf("<p><strong>USER:</strong> %s</p>\n", 
           user ? user : "Not Set");
    
    // Show all environment variables (both CGI and regular)
    printf("<h2>All Environment Variables</h2>\n");
    printf("<pre>\n");
    
    // extern char **environ is available on most Unix systems
    extern char **environ;
    for (int i = 0; environ[i] != NULL; i++) {
        // Only show first 50 variables to avoid overwhelming output
        if (i < 50) {
            printf("%s\n", environ[i]);
        } else {
            printf("... (truncated)\n");
            break;
        }
    }
    
    printf("</pre>\n");
    printf("</body></html>\n");
    
    return 0;
}
```

# **Flashcards:**
---

What is the primary difference between CGI environment variables and regular Linux environment variables in terms of scope and lifetime?;; CGI environment variables exist only during the execution of a single CGI script and are created fresh for each HTTP request, while regular Linux environment variables persist across shell sessions and are inherited by child processes.

How do HTTP headers get converted into CGI environment variables?;; HTTP headers are converted by adding an "HTTP_" prefix, converting to uppercase, and replacing hyphens with underscores. For example, "User-Agent" becomes "HTTP_USER_AGENT" and "Content-Type" becomes "CONTENT_TYPE" (special case).

Name four standard CGI environment variables and what information they contain.;; REQUEST_METHOD (HTTP method like GET/POST), QUERY_STRING (URL parameters after ?), SERVER_NAME (web server hostname), REMOTE_ADDR (client's IP address), CONTENT_TYPE (MIME type of request body), CONTENT_LENGTH (size of request body).

Why can't regular user environment variables like HOME or USER be reliably used in CGI scripts?;; CGI scripts typically run under the web server's user account (like www-data or apache) rather than a regular user account, so variables like HOME and USER either don't exist or contain values for the web server user, not the original client user.

How does the web server populate CGI environment variables, and when does this happen?;; The web server automatically populates CGI environment variables based on the incoming HTTP request before launching the CGI script process. This happens for each request and includes parsing HTTP headers, server configuration, and client information.

What happens to CGI environment variables after a CGI script finishes execution, and how does this differ from regular environment variables?;; CGI environment variables are destroyed when the CGI process terminates after handling the request, as they exist only in that process's memory space. Regular environment variables persist in their parent shell or system process and can be inherited by other processes.
