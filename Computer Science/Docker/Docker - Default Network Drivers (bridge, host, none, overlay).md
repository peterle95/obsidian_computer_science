---
memory: to_finish
tags:
  - to_learn
language:
  - Docker
review-date: 2025-11-20
last-reviewed: ""
scheda: done
visit-count: 0
confidence-level: 1
consecutive-correct: 0
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

By default, Docker containers are isolated from each other and the host machine. While this isolation is a key security feature, applications often require containers to communicate with each other, with the host, or with external networks. Docker's network drivers solve this problem by providing a pluggable system to define how containers connect and communicate.

Choosing the right network driver is crucial for application performance, security, and architecture. It allows developers to create everything from simple, single-host applications to complex, multi-host clusters, ensuring that containers can interact in a predictable and secure manner. This system abstracts away the underlying network complexities, enabling developers to focus on the application logic.

# **Core Explanation:**
---

Docker's networking model uses drivers to provide different types of network configurations for containers. Each driver is suited for a different use case, determining the level of isolation and the method of communication. The four default and most common drivers are:
## 1. Bridge

- **What it is:** The default network driver for containers. It creates a private, internal network on the host machine. Containers on the same bridge network can communicate with each other using their internal IP addresses or container names (if on a user-defined bridge).

- **How it works:** Docker creates a virtual network bridge ( docker0 by default) on the host. Each container connected to this network gets its own IP address from the bridge's IP range. For external communication, Docker uses Network Address Translation (NAT) and port mapping to forward traffic from the host's network to the appropriate container.

- **Primary Use:** Applications running in standalone containers that need to communicate on the same host. It's the most common driver for local development environments.

## 2. Host

- **What it is:** This driver removes network isolation between the container and the Docker host. The container effectively shares the host's network stack.

- **How it works:** The container does not get its own IP address. Instead, it uses the host's IP and network interfaces directly. Any port a containerized application listens on is directly exposed on the host machine without needing port mapping.

- **Primary Use:** When network performance is critical and network isolation is not a concern. It's useful for applications that need to handle a large range of ports or for network monitoring tools.

## 3. None

- **What it is:** This driver completely disables networking for a container, placing it in a network-isolated environment.

- **How it works:** The container is not attached to any network. It only receives a loopback interface (for processes inside the container to communicate with each other) and has no external connectivity.

- **Primary Use:** For containers that do not require any network access, such as batch processing jobs, or for security-sensitive applications where complete network isolation is required.

## 4. Overlay

- **What it is:** The overlay driver creates a distributed network that can span multiple Docker hosts. It is the primary networking solution for multi-host container communication.

- **How it works:** It creates a virtual network that "overlays" on top of the host networks, using VXLAN technology to encapsulate traffic and allow containers on different hosts to communicate as if they were on the same network. It is typically used with Docker Swarm for service discovery and load balancing.

- **Primary Use:** Multi-host applications and Docker Swarm services that need to communicate across different machines.

# **Related Concepts:**
---

- **Docker Swarm:** A container orchestration tool (similar to Kubernetes) that manages a cluster of Docker hosts.

 - **Connection:** The overlay network driver is the default and integral networking solution for Docker Swarm, enabling services to communicate across the different nodes in the swarm.

- **User-Defined Networks:** While Docker provides a default bridge network, users can create their own networks (e.g., docker network create my-net).

 - **Connection/Difference:** User-defined bridge networks are superior to the default one because they provide automatic DNS resolution between containers using their names, which the default bridge does not. They also offer better isolation.

- **Port Mapping (Port Forwarding):** The process of mapping a port on the host machine to a port inside a container.

 - **Connection:** This is essential for the bridge driver to allow external traffic to reach a container. With the host driver, port mapping is unnecessary as the container uses the host's ports directly.

- **Macvlan Driver:** A less common default driver that allows you to assign a MAC address to a container, making it appear as a physical device on your network.

 - **Difference:** While bridge networks isolate containers on an internal network, macvlan connects them directly to the physical network, which is useful for legacy applications that expect to be on the network directly.

# **Examples:**
---

```bash

#
---
1. Bridge Network (Default and User-Defined)
---
# By default, a container uses the 'bridge' network.

# These two containers are on the default bridge and can communicate via IP, but not by name.
docker run -d --name nginx1 nginx
docker run -d --name nginx2 nginx

# Create a user-defined bridge network for better features like DNS resolution.
docker network create my-bridge-net

# Run containers on the user-defined network.
docker run -d --name nginx3 --network my-bridge-net nginx
docker run -d --name nginx4 --network my-bridge-net nginx

# Now, nginx3 can ping nginx4 by its container name.

# This demonstrates the automatic DNS resolution of user-defined bridge networks.
docker exec nginx3 ping nginx4

#
---
2. Host Network
---
# Run a container that shares the host's network stack.

# The '--network host' flag activates this driver.

# If nginx listens on port 80 inside the container, it's now accessible on port 80 of the host machine.

# No '-p 80:80' port mapping is needed.
docker run -d --name nginx-host --network host nginx

#
---
3. None Network
---
# Run a container with all networking disabled.

# This container has no external IP address and cannot connect to the internet or other containers.

# The '--network none' flag is used.
docker run --rm --network none alpine ip addr show

# The output will only show the 'lo' (loopback) interface, not an 'eth0' interface.

#
---
4. Overlay Network (Requires Docker Swarm)
---
# First, initialize Docker Swarm on the host. This is a prerequisite for overlay networks.
docker swarm init

# Create an overlay network. The '--driver overlay' specifies the driver type.

# The '--attachable' flag allows standalone containers (not just swarm services) to connect to it.
docker network create --driver overlay --attachable my-overlay-net

# Run a service on one host using the overlay network.

# (On a multi-host setup, you would join other nodes to this swarm)
docker run -d --name overlay-container1 --network my-overlay-net alpine sleep 3600

# On another host (or the same one for this example), run another container on the same network.
docker run -d --name overlay-container2 --network my-overlay-net alpine sleep 3600

# These two containers can now communicate directly with each other,

# even if they were running on different physical machines in the swarm.
docker exec overlay-container1 ping overlay-container2
```

# **Flashcards:**
---

What is the default network driver in Docker, and what is its main purpose?;;**Bridge**. Its purpose is to create a private, internal network on a single host, allowing containers on that network to communicate while being isolated from external networks.

What is the key characteristic of the host network driver?;;It removes network isolation between the container and the host. The container shares the host's network stack and IP address, eliminating the need for port mapping.

In which scenario would you use the none network driver?;;When a container requires complete network isolation and should not have any connectivity to other containers or the external network.

What problem does the overlay network driver solve?;;It enables communication between containers running on **different Docker hosts**, creating a distributed network that spans multiple machines, and is essential for Docker Swarm.

Why is a user-defined bridge network generally preferred over the default bridge network?;;User-defined bridge networks provide automatic DNS resolution between containers, allowing them to communicate using their names, and offer better network isolation.

What technology does the overlay driver use to enable multi-host communication?;;It uses **VXLAN** (Virtual Extensible LAN) to create a virtual network that encapsulates traffic, allowing it to be routed across different host networks.