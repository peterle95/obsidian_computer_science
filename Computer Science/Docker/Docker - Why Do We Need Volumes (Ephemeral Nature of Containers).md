---
memory: to_finish
tags:
  - to_learn
language:
  - Docker
review-date: 2025-11-20
last-reviewed: 2025-10-23
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-09-30
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

==The fundamental problem Docker volumes solve is the **ephemeral nature of containers**. By default, a container's file system is temporary. When a container is removed, all data created inside it—such as application logs, user uploads, or database files—is permanently deleted. This makes running **stateful applications** (like databases) impossible without a solution.==

==The primary application of volumes is to **persist data** beyond the lifecycle of a single container.== Volumes act as a durable storage location that lives on the host machine, separate from the container itself. This is crucial because it allows you to stop, remove, or update a container without losing the application's critical data. Volumes are the standard, officially recommended way to manage persistent data in Docker.

# Core Explanation:
---

A **Docker ==volume** is a special directory on the host machine that is managed by Docker and can be mounted into a container's filesystem==. It provides a way to decouple the data from the container, ensuring the data persists.

### The Problem: The Ephemeral Container Layer Layers

A container's filesystem is built from a series of read-only image layers. When a container is started, Docker adds a thin, writable layer on top, called the **container layer**. Any changes made inside the container (creating files, modifying configurations) are written to this top layer. The core issue is that when the container is deleted (e.g., with `docker rm`), this writable container layer is destroyed, and all the data within it is lost.

### The Solution: Detaching Data with Volumes 💾

A<mark style="background: #ADCCFFA6;"> volume bypasses the container's layered filesystem. When you mount a volume into a container</mark> (e.g., mapping a volume to `/var/lib/mysql` inside a MySQL container), <mark style="background: #ADCCFFA6;">you are creating a direct link from that in-container path to a folder managed by Docker on the host machine.</mark>

Key characteristics:

- **Managed by Docker:** Docker handles the creation, management, and location of the volume's data on the host. This abstracts away the complexity of the host's operating system.
    
- **Persistent:** The data in a volume survives even if the container using it is removed.
    
- **Shareable:** The same volume can be mounted into multiple containers simultaneously, allowing them to share data.
    
- **Performant:** Volumes are often more performant for heavy write operations than the container's writable layer.
    

<mark style="background: #ADCCFFA6;">When the container writes to the mounted path</mark> (e.g., `/var/lib/mysql`), <mark style="background: #ADCCFFA6;">it's actually writing directly to the directory on the host machine. If you remove the container and launch a new one with the same volume attached, the new container will see and be able to use all the data saved by the original one.</mark>

# Related Concepts:
---

- **Stateless vs. Stateful Applications:** This is the core distinction that makes volumes necessary. A **stateless** app (e.g., a web server serving static files) doesn't need to save data between sessions. A **stateful** app (e.g., a database) must save data, and therefore requires a persistence mechanism like volumes.
    
- **Bind Mounts:** An alternative to volumes. A bind mount maps a _specific_ file or directory from the host machine into a container (the path is controlled by the user, not Docker). While useful for development (e.g., mounting your source code into a container for live reloading), **volumes are the preferred choice for application data** because they are managed by Docker, are more cross-platform compatible, and don't rely on a rigid host directory structure.
    
- **Image Layers:** The read-only building blocks of a container's filesystem. Understanding that these layers are immutable and that the container's writable layer is temporary is key to understanding _why_ data is lost and why volumes are needed to circumvent this.
    
- **Data Persistence:** The general computer science concept of information surviving after the process that created it has ended. Docker volumes are Docker's primary implementation of data persistence.
    

# Examples:
---

### Example 1: The Problem (Data Loss in an Ephemeral Container)

```bash
# 1. Run a temporary Ubuntu container. The '--rm' flag automatically removes it when we exit.
#    We use '-it' for an interactive terminal session.
docker run -it --rm ubuntu

# --- Now, you are INSIDE the container's shell ---
# 2. Create a directory and a file inside it. This file is written to the container's top writable layer.
mkdir /mydata
echo "This data is temporary" > /mydata/test.txt
cat /mydata/test.txt  # This will show the text.

# 3. Exit the container. The '--rm' flag now deletes it.
exit
# --- You are now back on your host machine ---

# 4. Try to start a new container and find the file.
docker run -it --rm ubuntu

# --- INSIDE the new container ---
# 5. The file is gone because the first container and its writable layer were destroyed.
ls /mydata  # This will result in an error: "ls: cannot access '/mydata': No such file or directory"
exit
```

### Example 2: The Solution (Persisting Data with a Volume)

```Bash
# 1. First, create a Docker-managed volume. We'll give it a name.
docker volume create my-app-data

# 2. Run a container and attach the volume.
#    The '-v' flag maps our volume 'my-app-data' to the '/mydata' directory inside the container.
#    We still use '--rm' to show that even when the container is deleted, the data will survive.
docker run -it --rm -v my-app-data:/mydata ubuntu

# --- Now, you are INSIDE the container ---
# 3. Create a file in the mounted directory. This data is being written to the volume on the host, NOT the container layer.
echo "This data will persist!" > /mydata/permanent-test.txt
cat /mydata/permanent-test.txt # This shows the text.

# 4. Exit the container. It gets automatically removed.
exit
# --- You are now back on your host machine ---

# 5. Run a brand new container, but attach the SAME volume to the SAME path.
docker run -it --rm -v my-app-data:/mydata ubuntu

# --- INSIDE the new container ---
# 6. Check for the file. It's still there! The data has persisted beyond the life of the first container.
cat /mydata/permanent-test.txt # It works! It outputs "This data will persist!"
exit
```

# Flashcards:
---

What does it mean for a Docker container to be "ephemeral"?;; It is temporary and stateless. Any data written directly to the container's filesystem is lost forever when the container is removed.1

What is the primary purpose of a Docker volume?;; To persist data generated by and used by containers, ensuring the data survives even after the container that created it is deleted.2

Where is the data in a Docker volume actually stored?;; On the host machine's filesystem, in a special directory that is fully managed by the Docker daemon.

What is the key difference between a Docker volume and a bind mount?;; Volumes are fully managed by Docker and are the preferred method for application data (databases, uploads). Bind mounts map a specific, user-defined host directory into the container, which is mainly used for mounting source code during development.

You run a PostgreSQL database in a container.3 How do you ensure its data is not lost if the container is removed or updated?;; By storing the database files in a Docker volume mounted to PostgreSQL's data directory (e.g., -v pg-data:/var/lib/postgresql/data).

What happens to the data inside a volume if you run docker rm my_container?;; Nothing. The data in the volume remains safe on the host machine. The volume is simply detached from the deleted container and can be attached to a new one.4