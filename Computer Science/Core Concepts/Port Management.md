---
memory: to_finish
tags:
 - learned
language:
 - Core Concepts
review-date: ""
last-reviewed: 2025-07-16
scheda: done
visit-count: 4
confidence-level: 1
consecutive-correct: 1
last-struggle-date: 2025-06-29

---
```dataviewjs
const currentPage = dv.current;
let visitCount = currentPage.file.frontmatter["visit-count"] || 0;
let confidence = currentPage.file.frontmatter["confidence-level"] || 1;
let streak = currentPage.file.frontmatter["consecutive-correct"] || 0;

const container = this.container.createEl('div');
container.style.cssText = `
 background:

# 2a2a2a; border: 1px solid

# 404040; border-radius: 6px;
 padding: 12px; margin: 10px 0; display: inline-block;
`;

// Status display
const status = container.createEl('div');
status.innerHTML = `
 <strong>Learning Progress</strong><br>
 Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
`;
status.style.cssText = 'margin-bottom: 10px; font-size: 13px; color:

# cccccc;';

// Quick feedback buttons
const buttonContainer = container.createEl('div');
['Got it! ✅', 'Struggled ⚠️', 'Failed ❌'].forEach((label, index) => {
 const btn = buttonContainer.createEl('button');
 btn.textContent = label;
 btn.style.cssText = `
 margin-right: 8px; padding: 4px 8px; border: none; border-radius: 3px;
 cursor: pointer; font-size: 11px;
 background: ${['

# 28a745', '

# ffc107', '

# dc3545'][index]};
 color: ${index === 1 ? '

# 000' : '

# fff'};
 `;

 btn.addEventListener('click', async => {
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
 fm["last-reviewed"] = new Date.toISOString.split('T');
 if (index > 0) fm["last-struggle-date"] = new Date.toISOString.split('T');
 });

 status.innerHTML = `
 <strong>Learning Progress</strong><br>
 Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
 `;
 });
});
```

# **Core Explanation:**

---
**Port management** refers to the process of assigning, monitoring, and resolving conflicts related to network ports on a computer system. Network ports are numerical labels (0-65535) that identify specific applications or services running on a server. When you run an application, it "listens" for incoming connections on a specific port. If another application is already using that port, you'll encounter an "Address already in use" error.

This is a very common issue in development, especially when working with multiple services (like a backend API and a Docker container) that might try to use the same default ports.

# **Related Concepts:**

---
- **IP Address:** Identifies a specific device on a network (e.g., `127.0.1` for localhost).
- **Port Number:** Identifies a specific application or service on that device.
- **Socket:** The combination of an IP address and a port number (e.g., `127.0.1:8000`).
- **Listening Port:** A port that an application is actively waiting for incoming connections on.
- **CORS (Cross-Origin Resource Sharing):** While not directly about port _conflicts_, CORS errors often arise when a frontend on one port (e.g., `3000`) tries to connect to a backend on a different port (e.g., `8001`) without proper backend configuration.

# **Examples:**

---
#

# **Scenario: InnoBee Backend "Address already in use"**

When trying to run the InnoBee Flask backend, we encountered the `Address already in use` error. The Flask app was configured to run on `.0.1:8000`, but something else was already listening on that port.

**Step-by-step resolution:**

1. **Initial Error:**

 ```
 Flask is running on .0.1:8000 (Debug: True)
 ...
 Address already in use
 Port 8000 is in use by another program.
 ```

 This clearly indicates a port conflict on `8000`.

2. **Attempting to change the port via environment variable (Failed):**

 ```bash
 FLASK_PORT=8001 python3 run.py
 ```

 _Explanation:_ We tried to tell the Flask app to use port `8001` by setting an environment variable `FLASK_PORT`. However, the Flask app's configuration (likely in `run.py` or `app.py`) was **hardcoded** to `8000` and didn't properly read this environment variable. The output continued to show it starting on `8000`.

3. **Identifying the process using port 8000:** To find out _what_ was using port `8000`, we used `sudo lsof -i :8000`.

 ```bash
 ubuntu@Peter-i9 ~/InnoBee-Backend (dev)> sudo lsof -i :8000
 COMMAND PID USER FD TYPE DEVICE SIZE/OFF NODE NAME
 docker-pr 1116 root 7u IPv4 4908 0t0 TCP *:8000 (LISTEN)
 docker-pr 1123 root 7u IPv6 4909 0t0 TCP *:8000 (LISTEN)
 ```

 _Explanation:_ The output showed `docker-pr` (likely `docker-proxy`) was listening on port `8000` for both IPv4 and IPv6. This means a Docker container was already running and exposing port `8000` on the host system.

4. **Stopping the conflicting Docker container:** Since the Docker container was the culprit, we needed to stop it.

 ```bash

# First, list running Docker containers to get their IDs
 docker ps

# Then, stop the specific container using port 8000 (replace <CONTAINER_ID>)
 docker stop <CONTAINER_ID>
 ```

 _Explanation:_ Stopping the Docker container frees up port `8000`.

5. **Configuring Flask to properly use the `FLASK_PORT` environment variable:** To make Flask dynamic, we needed to modify the `run.py` (or `app.py`) file.

 ```python
 import os
 from app import app

# Assuming your Flask app instance is named 'app'

# ... other imports and app setup ...

 if __name__ == '__main__':

# Get the port from the environment variable FLASK_PORT,

# defaulting to 8000 if not set.
 port = int(os.environ.get('FLASK_PORT', 8000))

# Run the Flask application on the specified port
 app.run(host='0.0.0', port=port, debug=True)

# Changed host to 0 for external access
 ```

 _Explanation:_ This code snippet tells Flask to read the `FLASK_PORT` environment variable. If `FLASK_PORT` is set (e.g., to `8001`), it will use that port; otherwise, it defaults to `8000`.

6. **Running the Flask backend successfully on a new port:** After stopping the Docker container and modifying the Flask code, we could finally run the backend on port `8001`.

 ```bash
 FLASK_PORT=8001 python3 run.py
 ```

 _Output:_

 ```bash
 ...
 2025-06-06 15:39:39,087 - INFO - Flask application initialized successfully
 Flask is running on .0.1:8001 (Debug: True)
 2025-06-06 15:39:39,088 - INFO - 🚀 Server starting on .0.1:8001
 * Serving Flask app 'app'
 * Debug mode: on
 ```

 _Explanation:_ Success! The backend is now listening on `8001`.

7. **Updating Frontend API Base URL:** Crucially, after the backend moved to `8001`, the frontend (running on `3000`) was still trying to connect to `8000`. This resulted in a **CORS error** later. The fix was to update the frontend's API base URL.

 ```typescript
 // In src/services/api/axiosInstance.ts (Frontend)
 // Change from:
 // baseURL: ".0.1:8000/api/v1",
 // To:
 baseURL: ".0.1:8001/api/v1",
 ```

 _Explanation:_ The frontend now correctly points to the backend's new port, resolving the connection issue.


---
EXAMPLE CODE

```bash

#
---
Identifying Processes on a Port
---
# Command 1: 'lsof -i :<PORT_NUMBER>' - Lists open files (and network connections)

# Syntax: lsof -i :<port_number>

# Requires sudo for full process information.

# Output shows COMMAND, PID, USER, and the NAME (IP:Port).
echo "Checking what's on port 8000 using lsof:"
sudo lsof -i :8000

# Example Output:

# COMMAND PID USER FD TYPE DEVICE SIZE/OFF NODE NAME

# docker-pr 1116 root 7u IPv4 4908 0t0 TCP *:8000 (LISTEN)

# Command 2: 'netstat -tulpn | grep :<PORT_NUMBER>' - Shows network connections and listening ports

# -t: TCP connections

# -u: UDP connections

# -l: Listening sockets

# -p: Show PID/Program name (requires sudo)

# -n: Numeric addresses (don't resolve hostnames)

# Syntax: netstat -tulpn | grep :<port_number>
echo -e "\nChecking what's on port 8000 using netstat:"
sudo netstat -tulpn | grep :8000

# Example Output:

# tcp 0 0 0.0.0:8000 0.0.0:* LISTEN 1116/docker-proxy

#
---

Killing a Process by PID
---
# Command: 'kill -9 <PID>' - Forcefully terminates a process.

# Use with caution, as it doesn't allow the process to clean up gracefully.

# Replace <PID> with the Process ID obtained from lsof or netstat.
echo -e "\nSimulating killing a process (assuming PID 12345 was found):"

# sudo kill -9 12345
echo " (Command 'sudo kill -9 12345' would be run here to kill the process with PID 12345)"

# Command: 'fuser -k <PORT_NUMBER>/tcp' - Kills all processes using a specific TCP port.

# This is often more convenient for port conflicts.
echo -e "\nSimulating killing processes on port 8000 using fuser:"

# sudo fuser -k 8000/tcp
echo " (Command 'sudo fuser -k 8000/tcp' would be run here to kill processes on port 8000)"

#
---

Running a Flask App on a Specific Port via Environment Variable
---
# Assuming your Flask app's run.py is configured to read FLASK_PORT
echo -e "\nRunning Flask app on port 8001 using FLASK_PORT environment variable:"
FLASK_PORT=8001 python3 your_flask_app/run.py

# If your Flask app supports a --port argument directly:

# python3 your_flask_app/run.py --port 8001

#
---

Example Flask app.run configuration for dynamic port
---
echo -e "\n
---

Example Python code in your Flask app (e.g., run.py)
---
"
cat <<EOF
import os
from flask import Flask

app = Flask(__name__)

# Your Flask app instance

if __name__ == '__main__':

# Get the port from the environment variable 'FLASK_PORT', default to 5000 if not set

# The 'int' conversion is crucial as environment variables are strings
 port = int(os.environ.get('FLASK_PORT', 5000))

# Run the Flask application

# 'host=0.0.0' makes the server accessible from other machines on the network,

# otherwise it defaults to '127.0.1' (localhost only).
 app.run(host='0.0.0', port=port, debug=True)
EOF
```

# **Flashcards:**

---
What command helps identify which process is using a specific network port (e.g., 8000) on Linux?;; `sudo lsof -i :8000` or `sudo netstat -tulpn | grep :8000`

My Flask app reports "Address already in use" on port 8000, even after setting `FLASK_PORT=8001`. Why?;; The Flask application's code is likely hardcoded to use port 8000 and isn't reading the `FLASK_PORT` environment variable. You need to modify the `app.run` call to `port = int(os.environ.get('FLASK_PORT', 8000))` (or similar).

After changing my backend port from 8000 to 8001, my frontend (on 3000) shows a CORS error. What's the immediate fix for the frontend?;; Update the frontend's API base URL configuration (e.g., in `axiosInstance.ts`) to point to the new backend port: `.0.1:8001/api/v1`.

test;; tests

asda;; hhshs

```dataviewjs
// Get all flashcards from the current note
const currentPage = dv.current;
const content = await app.vault.read(app.vault.getAbstractFileByPath(currentPage.file.path));

// Split content into lines
const lines = content.split('\n');
let flashcardLines = ;
let inCodeBlock = false;

// Collect all potential flashcard lines - simplified approach
for (let i = 0; i < lines.length; i++) {
 const line = lines[i];

 // Track code blocks
 if (line.trim.startsWith('```')) {
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

// Remove only the actual script lines, not user content
const filteredLines = flashcardLines.filter(line => {
 // Only filter out lines that are clearly part of the JavaScript code
 return !(
 line.includes('content.split') ||
 line.includes('flashcardLines') ||
 line.includes('dataviewjs') ||
 line.includes('const ') ||
 line.includes('let ') ||
 line.includes('function ') ||
 line.includes('.map(') ||
 line.includes('.filter(') ||
 line.includes('.forEach(') ||
 line.includes('.find(') ||
 line.includes('console.log') ||
 line.includes('return ') ||
 line.includes('if (') ||
 line.includes('for (') ||
 line.includes('while (') ||
 line.includes('=>') ||
 line.includes('this.container') ||
 line.includes('addEventListener')
 );
});

// Process the flashcards
const flashcards = ;
for (let i = 0; i < filteredLines.length; i++) {
 const line = filteredLines[i];
 try {
 const separatorIndex = line.indexOf(';;');
 if (separatorIndex === -1) continue;

 const front = line.substring(0, separatorIndex).trim;
 const back = line.substring(separatorIndex + 2).trim;

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
 errorMsg.style.cssText = 'background:

# 2a2a2a; padding: 15px; border-radius: 6px; color:

# cccccc;';
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
 background:

# 2a2a2a;
 border: 1px solid

# 404040;
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
title.style.cssText = 'margin: 0; color:

# ffffff;';

const progress = header.createEl('div');
progress.style.cssText = 'color:

# cccccc; font-size: 14px; text-align: right;';

// Card container
const cardContainer = container.createEl('div');
cardContainer.style.cssText = `
 background:

# 1a1a1a;
 border: 2px solid

# 404040;
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
 color:

# ffffff;
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
 background:

# 4a9eff; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; font-weight: 500;
`;

const easyButton = buttonContainer.createEl('button');
easyButton.textContent = 'Easy ✅';
easyButton.style.cssText = `
 background:

# 28a745; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const hardButton = buttonContainer.createEl('button');
hardButton.textContent = 'Hard ❌';
hardButton.style.cssText = `
 background:

# dc3545; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const nextButton = buttonContainer.createEl('button');
nextButton.textContent = 'Next →';
nextButton.style.cssText = `
 background:

# 6c757d; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const prevButton = buttonContainer.createEl('button');
prevButton.textContent = '← Prev';
prevButton.style.cssText = `
 background:

# 6c757d; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const shuffleButton = buttonContainer.createEl('button');
shuffleButton.textContent = '🔀 Shuffle';
shuffleButton.style.cssText = `
 background:

# 17a2b8; color: white; border: none; border-radius: 4px;
 padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

// Functions
function updateDisplay {
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
 cardContainer.style.borderColor = '

# ffc107';
 cardContainer.style.backgroundColor = '

# 252525';
 } else {
 easyButton.style.display = 'none';
 hardButton.style.display = 'none';
 flipButton.textContent = 'Flip Card';
 cardContainer.style.borderColor = '

# 404040';
 cardContainer.style.backgroundColor = '

# 1a1a1a';
 }

 // Update navigation buttons
 prevButton.style.display = currentCardIndex > 0 ? 'inline-block' : 'none';
 nextButton.textContent = currentCardIndex < flashcards.length - 1 ? 'Next →' : 'Restart';
}

function flipCard {
 showingBack = !showingBack;
 updateDisplay;
}

function nextCard {
 if (currentCardIndex < flashcards.length - 1) {
 currentCardIndex++;
 } else {
 currentCardIndex = 0;
 }
 showingBack = false;
 updateDisplay;
}

function prevCard {
 if (currentCardIndex > 0) {
 currentCardIndex--;
 showingBack = false;
 updateDisplay;
 }
}

function markCorrect {
 if (showingBack) {
 correctCount++;
 totalReviewed++;
 nextCard;
 }
}

function markIncorrect {
 if (showingBack) {
 totalReviewed++;
 nextCard;
 }
}

function shuffleCards {
 for (let i = flashcards.length - 1; i > 0; i--) {
 const j = Math.floor(Math.random * (i + 1));
 [flashcards[i], flashcards[j]] = [flashcards[j], flashcards[i]];
 }
 currentCardIndex = 0;
 showingBack = false;
 correctCount = 0;
 totalReviewed = 0;
 updateDisplay;
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
instructions.style.cssText = 'font-size: 12px; color:

# 888; text-align: center; line-height: 1.4;';
instructions.innerHTML = `
 <strong>Controls:</strong> Click card to flip | Navigation buttons | Easy/Hard to mark
`;

// Initialize
updateDisplay;
```