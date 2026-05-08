---
memory: to_finish
tags:
 - learned
language:
 - Docker
review-date: ""
last-reviewed: 2025-08-06
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 3

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

const filteredLines = flashcardLines.filter(line => {
 // Only filter out lines that are clearly part of the JavaScript code
 // Be more specific with patterns to avoid false positives
 return !(
 line.trim.startsWith('const ') ||
 line.trim.startsWith('let ') ||
 line.trim.startsWith('function ') ||
 line.trim.startsWith('return ') ||
 line.trim.startsWith('if (') ||
 line.trim.startsWith('for (') ||
 line.trim.startsWith('while (') ||
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

# **Core Explanation:**

---
This note focuses on advanced Docker commands essential for managing and troubleshooting Docker containers, especially in development environments where you're running multiple services or encountering conflicts. While `docker build` and basic `docker run` are used for initial setup, commands like **`docker ps`**, **`docker stop`**, and utilities like **`lsof`**, **`netstat`**, and **`fuser`** become crucial for debugging runtime issues such as port conflicts.

Understanding how to explicitly map ports, pass environment variables with **`--env-file`**, and diagnose processes using network ports is key to smooth Dockerized development.

# **Related Concepts:**

---
- **Docker Containers**: Running instances of Docker images.
- **Docker Images**: The blueprints from which containers are created.
- **Dockerfile**: Instructions for building Docker images.
- **Port Management**: The concept of network ports and how applications use them.
- **Environment Variables**: Dynamic named values that can affect the way running processes behave.
- **Debugging Strategies**: General approaches to identifying and resolving software issues.

# **Examples:**

---
#

#

# **1. Building a Docker Image (Review)**

This command was discussed in your [[Docker - What are Docker Images]] note but is fundamental for context.

```bash
docker build -t innobee-backend .

# -t innobee-backend: Tags the image with the name 'innobee-backend'. This makes it easy to reference later.

# .: Specifies the build context, which is the current directory (where your Dockerfile is located).

# Explanation: This command reads the Dockerfile in the current directory and builds a Docker image

# named 'innobee-backend'. This process installs dependencies (like Python packages)

# and sets up the environment as defined in the Dockerfile.
```

#

#

# **2. Running a Docker Container**

This is where we faced initial issues due to the missing `.env` file and later port conflicts.

```bash
docker run -p 8000:8000 --env-file .env innobee-backend

# -p 8000:8000: This is a crucial port mapping flag.

# - The first 8000 is the HOST port (your machine's port).

# - The second 8000 is the CONTAINER port (the port the Flask app inside Docker listens on, as defined in your Dockerfile, typically via EXPOSE or app.run).

# --env-file .env: Tells Docker to read environment variables from the specified '.env' file

# and pass them into the running container. This is vital for database URLs,

# secret keys, and other configurations.

# innobee-backend: The name of the Docker image to run.

# Explanation: This command starts a new container from the 'innobee-backend' image,

# maps port 8000 on your host machine to port 8000 inside the container, and injects

# environment variables from your '.env' file.
```

#

#

# **3. Listing Running Docker Containers (`docker ps`)**

Essential for checking what's currently active.

```bash
docker ps

# Explanation: Lists all currently running Docker containers.

# This is useful to see if your 'innobee-backend' container is actually running,

# and which ports it's mapped to on your host machine.

# Look for the 'PORTS' column to confirm port mappings.
```

#

#

# **4. Stopping a Running Docker Container (`docker stop`)**

To free up ports or restart a service.

```bash
docker stop <CONTAINER_ID>

# <CONTAINER_ID>: Replace this with the actual ID of the container you want to stop.

# You can get this ID from the 'CONTAINER ID' column when you run 'docker ps'.

# Explanation: Gracefully stops the specified running container.
```

```bash
docker stop $(docker ps -q)

# $(docker ps -q): This is a sub-command that finds the IDs of all running containers (-q for quiet, showing only IDs).

# Explanation: Stops all currently running Docker containers. Use with caution!
```

#

#

# **5. Diagnosing Port Conflicts on the Host Machine (`lsof`, `netstat`)**

These commands help identify which non-Docker process might be using a port, or if a previous Docker run left a process behind.

```bash
sudo lsof -i :8000

# sudo: Runs the command with superuser privileges, often necessary to see all network processes.

# lsof: Stands for "list open files".

# -i :8000: Filters the output to show processes that are listening on or connected to port 8000.

# Explanation: This command helped us identify that 'docker-pr' (Docker Proxy) was using port 8000.

# It tells you the COMMAND (program name), PID (Process ID), USER, and NODE (port being used).
```

```bash
sudo netstat -tulpn | grep :8000

# sudo: Superuser privileges.

# netstat: Network statistics.

# -t: Show TCP connections.

# -u: Show UDP connections.

# -l: Show listening sockets.

# -p: Show the PID and program name (requires sudo).

# -n: Show numerical addresses (don't try to resolve hostnames).

# | grep :8000: Pipes the output of 'netstat' to 'grep' to filter lines containing ':8000'.

# Explanation: Another powerful command to see what processes are listening on a specific port.

# It also provides PID and program name.
```

#

#

# **6. Killing a Process Using a Port (`kill`, `fuser`)**

Once you've identified the PID, you can terminate it.

```bash
sudo kill -9 <PID>

# sudo: Superuser privileges.

# kill: Sends a signal to a process.

# -9: The SIGKILL signal, which forcefully terminates a process immediately. Use with caution, as it doesn't allow the process to clean up gracefully.

# <PID>: Replace with the Process ID obtained from 'lsof' or 'netstat'.

# Explanation: This command will forcefully stop the process identified by <PID> that is holding the port.
```

```bash
sudo fuser -k 8000/tcp

# sudo: Superuser privileges.

# fuser: Identifies processes using files or sockets.

# -k: Kills the processes found.

# 8000/tcp: Specifies the TCP port 8000.

# Explanation: A more direct way to kill processes listening on a specific TCP port. It will find and kill them.
```

#

#

# **7. Passing Environment Variables to a Python/Flask App (Non-Docker)**

This was attempted initially before realizing Docker was holding the port. It's still a valid method for non-Docker runs.

```bash
FLASK_PORT=8001 python3 run.py

# FLASK_PORT=8001: Sets the environment variable FLASK_PORT to 8001 for this command only.

# python3 run.py: Executes your Flask application script.

# Explanation: Your Flask application's `app.run` function needs to be configured to

# read this environment variable (e.g., `port = int(os.environ.get('FLASK_PORT', 8000))`).

# If the app is hardcoded to 8000, setting an environment variable won't override it directly.
```

#

#

# **InnoBee Project Context:**

In our InnoBee project, we encountered the "Address already in use" error when trying to run the `InnoBee-Backend` Flask application directly. Using **`sudo lsof -i :8000`** revealed that `docker-pr` (Docker Proxy) was already listening on port 8000, indicating a previous Docker container was still holding the port. This led us to investigate Docker processes.

After successfully stopping the conflicting Docker process or modifying the Flask app to use `FLASK_PORT=8001` (and ensuring the app itself correctly reads this variable via `os.environ.get`), the backend successfully started, proving the importance of these advanced port and process management commands.

# **Flashcards:**

---
What Docker command lists all running containers?;; `docker ps`

How do you map host port 8001 to container port 8000 when running a Docker container?;; `docker run -p 8001:8000 <image_name>`

What command can you use to find out which process is using a specific port on your Linux host?;; `sudo lsof -i :<port_number>` or `sudo netstat -tulpn | grep :<port_number>`