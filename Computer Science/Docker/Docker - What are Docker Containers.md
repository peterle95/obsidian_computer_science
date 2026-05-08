---
memory: to_finish
tags:
  - learning
language:
  - Docker
review-date: 2025-11-25
last-reviewed: 2025-10-16
scheda: done
visit-count: 2
confidence-level: 2
consecutive-correct: 2
last-struggle-date: ""
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

Docker containers solve the fundamental problem of application consistency across different computing environments. It tackles the classic "it works on my machine" issue that developers often face. By packaging an application with all its necessary dependencies—such as libraries, system tools, code, and runtime—into a single, standardized unit, containers ensure that the application runs quickly and reliably regardless of the deployment environment. 

This is crucial in modern software development for several reasons:
*   **Portability:** Containers encapsulate an application and its dependencies, allowing them to be moved seamlessly between different environments, such as a developer's laptop, a testing server, or a production cloud environment. 
*   **Consistency:** It guarantees that the application will run the same way everywhere, which simplifies development, testing, and deployment cycles. 
*   **Efficiency:** Containers are lightweight because they share the host system's operating system kernel, leading to faster startup times and more efficient use of system resources compared to traditional virtual machines. 
*   **Isolation:** Applications running in containers are isolated from each other and the host system, which enhances security and prevents conflicts between dependencies. 

# **Core Explanation:**
---


A Docker container is a lightweight, standalone, executable package of software that includes everything needed to run an application. ==It is a running instance of a **Docker image**==. 

**Key Characteristics:**

*   **Standardized:** Docker created the industry standard for containers, ensuring they are portable and can run anywhere Docker is installed. 
*   **Lightweight:** Containers share the host machine's operating system kernel and do not require a separate guest OS for each application. This results in smaller image sizes (typically tens of MBs) and less overhead. 
*   **Isolated:** Containers run in isolated processes in the user space of the host operating system. Docker uses Linux kernel features like `namespaces` to provide this isolation, ensuring that each container has its own private view of the filesystem, network, and process tree. 
*   **Secure:** The isolation provided by containers means that applications are safer, and Docker offers strong default isolation capabilities. 

**How it Works:**

==Docker utilizes a client-server architecture. The Docker client communicates with the Docker daemon (the server), which does the heavy lifting of building, running, and distributing Docker containers. When you run a container, Docker takes an image and creates a runnable instance from it. This container includes the application and all its dependencies but shares the kernel of the host operating system.==

# **Related Concepts:**
---


*   **Docker Image:** A read-only template used to create Docker containers. It contains the application code, libraries, dependencies, and other files needed to run the application. Images are often based on other, more basic images, and are built using a `Dockerfile`. A container is a running instance of an image. 

*   **Dockerfile:** A text document that contains all the commands a user could call on the command line to assemble an image. It's essentially the blueprint for building a Docker image, specifying the base image, commands to run, files to copy, and more. 

*   **Virtual Machines (VMs):** While similar in their goal of providing isolated environments, VMs and containers differ fundamentally in their architecture. A VM virtualizes the hardware, meaning each VM includes a full copy of an operating system, the application, necessary binaries, and libraries. This makes them much larger and slower to start than containers, which only virtualize the operating system and share the host kernel. 

*   **Docker Compose:** A tool for defining and running multi-container Docker applications. It uses a YAML file to configure the application's services, making it easy to spin up complex applications with multiple interconnected containers (like a web server, database, and caching service) with a single command.

*   **Kubernetes:** A container orchestration platform used to automate the deployment, scaling, and management of containerized applications. While Docker provides the container runtime, Kubernetes takes care of managing those containers at scale across a cluster of machines.

# **Examples:**
---


Here is an example of a `Dockerfile` for a simple [[Node.js - Server-Side JavaScript]] web application.

**1. The Node.js application (`app.js`):**
```javascript
// A simple Express.js web server
const express = require('express');
const app = express();
const PORT = 8080;

app.get('/', (req, res) => {
  res.send('Hello, Docker!');
});

app.listen(PORT, () => {
  console.log(`Server is running on http://localhost:${PORT}`);
});```

2. The package.json file:
```json
{
  "name": "docker-hello-world",
  "version": "1.0.0",
  "description": "A simple Node.js app for Docker",
  "main": "app.js",
  "dependencies": {
    "express": "^4.17.1"
  }
}
```
3. The Dockerfile:

```dockerfile
# --- Base Image ---
# Use an official Node.js runtime as the base image.
# The '18-alpine' tag provides a lightweight version of Node.js.
FROM node:18-alpine

# --- Set Working Directory ---
# Create and set the working directory inside the container.
# This is where the application code will live.
WORKDIR /usr/src/app

# --- Copy Application Files ---
# Copy the package.json and package-lock.json files to the working directory.
# This is done separately to leverage Docker's layer caching.
# The dependencies will only be re-installed if these files change.
COPY package*.json ./

# --- Install Dependencies ---
# Run the 'npm install' command to install the application's dependencies.
RUN npm install

# --- Copy Source Code ---
# Copy the rest of the application's source code into the container.
COPY . .

# --- Expose Port ---
# Inform Docker that the container will listen on port 8080 at runtime.
EXPOSE 8080

# --- Define Startup Command ---
# Specify the command to run when the container starts.
# This will start the Node.js application.
CMD [ "node", "app.js" ]
```
4. How to build and run the container:
```bash
# Command to build the Docker image from the Dockerfile.
# The '-t' flag tags the image with a name (e.g., 'my-node-app').
# The '.' indicates that the Dockerfile is in the current directory.
docker build -t my-node-app .

# Command to run the Docker container from the newly created image.
# The '-p' flag maps port 8080 on the host to port 8080 in the container.
# The '-d' flag runs the container in detached mode (in the background).
docker run -p 8080:8080 -d my-node-app
```

# Flashcards:
---

What is a Docker Container?;; A lightweight, standalone, executable package of software that includes everything needed to run an application: code, runtime, system tools, libraries, and settings. It's a running instance of a Docker image.

What is the main problem Docker containers solve?;; They solve the "it works on my machine" problem by ensuring an application runs consistently and reliably across different computing environments.

How do Docker containers differ from Virtual Machines (VMs)?;; Containers virtualize the operating system and share the host OS kernel, making them lightweight. VMs virtualize the hardware and each VM runs a full, separate guest OS, making them much larger and more resource-intensive.

What is a Docker Image?;; A read-only template containing the application code, libraries, and dependencies. It is used as the blueprint to create a running Docker container. 

What is a Dockerfile?;; A text script with a series of instructions on how to build a Docker image automatically.

Why are containers considered "lightweight"?;; Because they share the host machine's operating system kernel and don't require their own full guest OS, which significantly reduces their size and resource consumption.
