---
memory: to_finish
tags:
  - learning
language:
  - Docker
review-date: 2025-11-25
last-reviewed: 2025-10-15
scheda: done
visit-count: 3
confidence-level: 1
consecutive-correct: 0
last-struggle-date: 2025-10-15
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

Docker networking addresses the fundamental challenge of ==enabling communication between isolated containers, and between containers and the external world==. <mark style="background: #BBFABBA6;">Containers, by design, are isolated environments</mark>. ==This isolation is a key security and organizational feature, but it also means that a structured way for them to communicate is necessary for them to work together to form a cohesive application==.

The primary applications of Docker networking are:

- **Enabling Microservices Architectures**: In a <mark style="background: #ABF7F7A6;">microservices architecture, an application is broken down into a collection of smaller, independently deployable services</mark>. Docker networking allows these services, each running in its own container, to communicate with each other.

- **Providing Service Discovery**: <mark style="background: #ABF7F7A6;">Docker has a built-in DNS service that allows containers on the same network to discover and communicate with each other using their service names</mark>.

- **Ensuring Application Portability and Scalability**: Docker's networking model allows containerized applications to be easily moved from one environment to another with consistent network configurations. It also simplifies the process of scaling applications by allowing new containers to be easily added to the network.

- **Enhancing Security through Isolation**: By default, containers on different networks cannot communicate with each other, providing a level of network segmentation and security.


In essence, Docker networking is crucial for building and running modern, distributed applications in a containerized environment. It provides the necessary tools to create flexible, scalable, and secure network configurations for containerized workloads.

# **Core Explanation:**
---

Docker networking refers to the system that allows Docker containers to communicate with each other, the host machine, and external networks. It provides a flexible and powerful way to manage the network stack for containers, enabling the creation of complex application topologies.

At its core, Docker networking is built on a set of network drivers. <mark style="background: #D2B3FFA6;">When you create a network, you specify which driver to use</mark>. The most common drivers are:

- **bridge**: <mark style="background: #D2B3FFA6;">The default network driver</mark>. It creates a <mark style="background: #D2B3FFA6;">private, internal network on the host.</mark> Containers on the same bridge network can communicate with each other.

- **host**: This driver removes network isolation between the container and the Docker host. The container shares the host's networking namespace.

- **overlay**: This driver is used for multi-host networking, enabling communication between containers running on different Docker hosts. It is commonly used with Docker Swarm.

- **macvlan**: This driver allows you to assign a MAC address to a container, making it appear as a physical device on your network.

- **none**: This driver disables all networking for a container.


**Key Characteristics:**

- **Isolation**: Docker provides network isolation, meaning containers on one network are isolated from containers on other networks unless explicitly connected.

- **Service Discovery**: Docker's built-in DNS server allows containers on the same user-defined network to resolve each other's names to their respective IP addresses.

- **Pluggable Drivers**: The networking subsystem is pluggable, allowing for the use of different network drivers to suit various use cases.

- **Container Network Model (CNM)**: This is the design specification for Docker networking. It defines the constructs that provide networking for containers, including Sandboxes, Endpoints, and Networks.


**How it Works:**

<mark style="background: #BBFABBA6;">When Docker is installed, it creates three default networks: bridge, host, and none</mark>. <mark style="background: #BBFABBA6;">For each container that is created, Docker assigns it a virtual network interface, an IP address from the network's subnet, a gateway, and DNS services</mark>. This allows the container to communicate with other containers on the same network and with the outside world. User-defined networks can also be created to provide better isolation and enable DNS-based service discovery between containers.

# **Related Concepts:**
---

- **Docker Swarm**: A container orchestration tool that is native to Docker. Docker Swarm uses overlay networks to facilitate communication between services running on different nodes in the swarm cluster.

- **[[Docker - Kubernetes (Introduction)]]**: Another popular container orchestration platform. While it manages container networking, it has its own networking model and uses different plugins (like Calico, Flannel) to implement it, which differs from Docker's default networking drivers.

- **[[Docker - Default Network Drivers (bridge, host, none, overlay)]]**: These are the pluggable components that provide the core networking functionality in Docker. The choice of network driver determines how containers will communicate.

- **Container Network Model (CNM)**: The design specification that Docker's networking implementation is based on. It provides a standard for how networking should be implemented for containers.

- **iptables**: A Linux utility that allows a system administrator to configure the IP packet filter rules of the Linux kernel firewall. Docker manipulates iptables rules on the host to provide network isolation and port mapping.

- **Network Namespaces**: A Linux kernel feature that provides isolation of the network stack. Each container gets its own network namespace, which includes its own network interfaces, IP addresses, routing tables, etc.

- [[Docker - Networking with Compose]]


# **Examples:**
---

```bash

# List all existing Docker networks

# This command shows the default networks (bridge, host, none) and any user-defined networks.
docker network ls

# Create a new user-defined bridge network

# It's a best practice to create custom bridge networks for your applications for better isolation and service discovery.
docker network create my-custom-network

# Inspect a network to see its details

# This command provides information about the network's driver, subnet, gateway, and connected containers.
docker network inspect my-custom-network

# Run a container and attach it to the custom network

# This will start an Nginx container named 'my-nginx-app' and connect it to 'my-custom-network'.
docker run -d --name my-nginx-app --network my-custom-network nginx

# Run another container on the same network

# This will start a busybox container named 'my-test-container' and attach it to the same network.
docker run -it --name my-test-container --network my-custom-network busybox

# From within the 'my-test-container', you can now ping the 'my-nginx-app' by its name

# This demonstrates the DNS-based service discovery provided by user-defined bridge networks.

# To do this, first, attach to the running busybox container:

# docker attach my-test-container

# Then, from the container's shell prompt, run:

# ping my-nginx-app

# Connect an already running container to a network

# First, run a container without attaching it to our custom network.
docker run -d --name another-container nginx

# Then, connect the running container to the 'my-custom-network'.
docker network connect my-custom-network another-container

# Disconnect a container from a network
docker network disconnect my-custom-network another-container

# Remove a custom network

# A network can only be removed if there are no containers connected to it.
docker network rm my-custom-network
```

# **Flashcards:**
---

What is the primary purpose of Docker networking?;; To enable communication between isolated Docker containers, the host machine, and external networks.

What are the five default network drivers in Docker?;; bridge, host, overlay, macvlan, and none.

What is the default network for a new container if not specified?;; The bridge network.

What is the main advantage of using a user-defined bridge network over the default bridge network?;; User-defined bridge networks provide better isolation and enable automatic DNS resolution between containers.

How can two containers on different hosts communicate with each other using Docker networking?;; By using an overlay network, which is designed for multi-host communication.

What command is used to see detailed information about a specific Docker network?;; docker network inspect <network_name>