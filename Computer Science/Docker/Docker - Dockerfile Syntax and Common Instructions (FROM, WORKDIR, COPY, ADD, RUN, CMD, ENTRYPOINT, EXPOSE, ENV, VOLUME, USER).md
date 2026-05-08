---
memory: to_finish
tags:
  - learning
language:
  - Docker
review-date: 2025-11-25
last-reviewed: 2025-10-09
scheda: done
visit-count: 7
confidence-level: 1.5
consecutive-correct: 1
last-struggle-date: 2025-09-25
cssclasses:
  - important
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
# **Core Explanation:**
---

Dockerfile syntax consists of a series of instructions that define how to build a Docker image. Each instruction creates a new layer in the image, following a specific format and execution model.

## General Syntax Rules:
- **Case-insensitive instructions**: FROM, from, and From are equivalent, but UPPERCASE is conventional
- **Line-by-line execution**: Instructions are processed sequentially from top to bottom
- **Comment syntax**: Lines starting with # are comments (except parser directives)
- **Instruction format**: `INSTRUCTION arguments`
- ==**Multi-line support**: Use backslash (\\) for line continuation==
- **Parser directives**: Special comments like `# syntax=docker/dockerfile:1` that modify parsing behavior

## Layer Creation and Caching:
- <mark style="background: #D2B3FFA6;">Each instruction creates a new read-only layer</mark>
- ==Layers are cached and reused if nothing has changed==
- ==Changes to an instruction invalidate cache for that layer and all subsequent layers==
- Layer caching significantly improves build performance

## Core Instructions Overview:

Here's your expanded Docker commands explanation with complete details:

## Core Docker Instructions Overview:

### **FROM**: Establishes the base image foundation

**Purpose**: Specifies the parent image that serves as the starting point for your Docker image. <mark style="background: #BBFABBA6;">Every Dockerfile must begin with a FROM instruction.</mark>

**Syntax**: `FROM <image>[:<tag>] [AS <name>]`

**Key Points**:

- Must be the first non-comment instruction in a Dockerfile
- Uses Docker Hub's official images or custom images as foundation
- Tag specification (e.g., `:latest`, `:alpine`, `:18`) determines the specific version
- Multi-stage builds use `AS <name>` to reference stages later

**Examples**:

```dockerfile
# Use official Node.js runtime as base
FROM node:18-alpine

# Use specific Ubuntu version
FROM ubuntu:22.04

# Multi-stage build naming
FROM node:18 AS builder
```

---

### **WORKDIR**: Sets the working directory context

**Purpose**: Establishes the current working directory inside the container <mark style="background: #BBFABBA6;">for subsequent instructions</mark>. Creates the directory if it doesn't exist.

**Syntax**: `WORKDIR /path/to/directory`

**Key Points**:

- <mark style="background: #BBFABBA6;">All relative paths in subsequent instructions are relative to this directory</mark>
- Equivalent to `cd` command but persistent across instructions
- Creates the directory automatically if it doesn't exist
- Can be used multiple times to change context

**Examples**:

```dockerfile
# Set working directory to /app
WORKDIR /app

# Change to subdirectory
WORKDIR /app/src

# Use absolute path
WORKDIR /var/www/html
```

---

### **COPY/ADD**: Transfers files from host to image

**Purpose**: Transfer files and directories from the build context (host machine) into the Docker image filesystem.

**Syntax**:

- `COPY <src>... <dest>`
- `ADD <src>... <dest>`

**Key Differences**:

- **COPY**: <mark style="background: #BBFABBA6;">Simple file copying, preferred for basic transfers</mark>
- **ADD**: Advanced features <mark style="background: #BBFABBA6;">including URL downloads and automatic tar extraction</mark>

**Key Points**:

- Source paths are relative to build context (directory containing Dockerfile)
- Destination can be absolute or relative to WORKDIR
- Preserves file permissions and metadata
- Supports wildcards and pattern matching

**Examples**:

```dockerfile
# Copy single file
COPY package.json /app/

# Copy directory contents
COPY src/ /app/src/

# Copy with wildcards
COPY *.js /app/scripts/

# ADD with URL (downloads file)
ADD https://example.com/file.tar.gz /app/

# ADD automatically extracts tar files
ADD archive.tar.gz /app/
```

---

### **RUN**: Executes commands during build process

**Purpose**: Executes commands inside the container during the image build process. <mark style="background: #BBFABBA6;">Each RUN instruction creates a new layer in the image.</mark>

**Syntax**:

- Shell form: `RUN <command>`
- Exec form: `RUN ["executable", "param1", "param2"]`

**Key Points**:

- Commands execute in the container's shell (`/bin/sh -c` by default)
- Each RUN creates a new image layer
- Chain commands with `&&` to reduce layers
- Use `\` for line continuation in long commands

**Examples**:

```dockerfile
# Install packages (shell form)
RUN apt-get update && apt-get install -y \
    curl \
    vim \
    git \
    && rm -rf /var/lib/apt/lists/*

# Install Node.js dependencies
RUN npm install

# Exec form (doesn't invoke shell)
RUN ["npm", "run", "build"]

# Multiple commands in one layer
RUN mkdir -p /app/logs && \
    chmod 755 /app/logs && \
    chown app:app /app/logs
```

---

### **CMD/ENTRYPOINT**: Defines container startup behavior

**Purpose**: <mark style="background: #FF5582A6;">Specify what command runs when the container starts</mark>. These instructions <mark style="background: #FF5582A6;">define the container's primary purpose.</mark>

**Key Differences**:

- **CMD**: Default command,<mark style="background: #D2B3FFA6;"> can be overridden by `docker run` arguments</mark>
- **ENTRYPOINT**: ==Fixed command, `docker run` arguments become parameters==

**Syntax**:

- Shell form: `CMD command param1 param2`
- Exec form: `CMD ["executable", "param1", "param2"]`

**Key Points**:

- ==Only the last CMD/ENTRYPOINT instruction takes effect==
- ==ENTRYPOINT + CMD work together: ENTRYPOINT is fixed, CMD provides default parameters==
- Exec form is preferred (doesn't invoke shell, handles signals properly)

**Examples**:

```dockerfile
# CMD - can be overridden
CMD ["npm", "start"]
CMD ["python", "app.py"]

# ENTRYPOINT - always runs
ENTRYPOINT ["python", "app.py"]

# Combined usage
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080"]  # Default parameters

# Shell form (not recommended for main process)
CMD npm start
```

---

### **EXPOSE**: Documents network ports

**Purpose**: Informs Docker (and users) ==which ports the container will listen on at runtime==. This is documentation only and doesn't actually publish ports.

**Syntax**: `EXPOSE <port> [<port>/<protocol>...]`

**Key Points**:

- <mark style="background: #BBFABBA6;"> Does NOT automatically publish ports to host</mark>
- <mark style="background: #BBFABBA6;">Serves as documentation for intended port usage</mark>
- Default protocol is TCP if not specified
- ==Use `docker run -p` to actually publish ports==

**Examples**:

```dockerfile
# Expose HTTP port
EXPOSE 80

# Expose multiple ports
EXPOSE 80 443

# Specify protocol
EXPOSE 53/udp
EXPOSE 80/tcp

# Application-specific ports
EXPOSE 3000 5432
```

---

### **ENV**: Sets environment variables

**Purpose**: Sets environment variables that persist in the container both during build and runtime.

**Syntax**:

- `ENV <key>=<value> ...`
- `ENV <key> <value>` (space-separated, single variable)

**Key Points**:

- Variables are available in subsequent build instructions
- Persist into running containers
- Can be overridden at runtime with `docker run -e`
- Supports variable expansion from previously defined ENV variables

**Examples**:

```dockerfile
# Single variable
ENV NODE_ENV=production

# Multiple variables
ENV APP_HOME=/app \
    APP_USER=appuser \
    APP_VERSION=1.0.0

# Using previously defined variables
ENV PATH=$APP_HOME/bin:$PATH

# Database configuration
ENV DB_HOST=localhost \
    DB_PORT=5432 \
    DB_NAME=myapp
```

---

### **VOLUME**: Creates mount points for external storage

**Purpose**: Creates mount points for externally mounted volumes, enabling persistent data storage and sharing between containers and host.

**Syntax**: `VOLUME ["/path/to/volume"]` or `VOLUME /path/to/volume`

**Key Points**:

- Creates anonymous volumes if no external volume is specified
- Data in volumes persists beyond container lifecycle
- Enables data sharing between containers
- Can be overridden with `docker run -v` for specific mounts

**Examples**:

```dockerfile
# Single volume
VOLUME ["/var/lib/mysql"]

# Multiple volumes
VOLUME ["/var/log", "/var/db"]

# Application data
VOLUME ["/app/data"]

# Configuration and logs
VOLUME ["/etc/myapp", "/var/log/myapp"]
```

---

### **USER**: Sets execution user context

**Purpose**: <mark style="background: #BBFABBA6;">Sets the user (and optionally user group) for running subsequent instructions and the final container process.</mark>

**Syntax**: `USER <user>[:<group>]` or `USER <UID>[:<GID>]`

**Key Points**:

- Improves security by <mark style="background: #BBFABBA6;">running processes as non-root</mark>
- User must exist in the container (create with RUN if needed)
- <mark style="background: #BBFABBA6;">Affects all subsequent RUN, CMD, and ENTRYPOINT instructions</mark>
- Can switch users multiple times during build

**Examples**:

```dockerfile
# Create and switch to non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser

# Use numeric IDs
USER 1001:1001

# Switch back to root for system tasks, then back to user
USER root
RUN apt-get update && apt-get install -y some-package
USER appuser

# Use existing system user
USER www-data

# Application-specific user setup
RUN adduser --disabled-password --gecos '' myapp
USER myapp:myapp
```

## Complete Dockerfile Example:

```dockerfile
# Multi-stage build example showing all instructions
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine AS runtime
ENV NODE_ENV=production \
    APP_USER=appuser
RUN addgroup -g 1001 -S $APP_USER && \
    adduser -S -D -H -u 1001 -h /app -s /sbin/nologin -G $APP_USER $APP_USER
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
RUN chown -R $APP_USER:$APP_USER /app
USER $APP_USER
EXPOSE 3000
VOLUME ["/app/data"]
ENTRYPOINT ["node"]
CMD ["server.js"]
```

## Execution Context:
- **Build context**: Directory containing Dockerfile and files to be included
- **Build stage**: Each FROM instruction starts a new build stage
- **Layer inheritance**: Each layer builds upon previous layers
- **Filesystem changes**: Instructions modify the filesystem incrementally

## Best Practices Principles:
- ==Minimize layers by combining related RUN commands==
- ==Place frequently changing instructions toward the end==
- Use specific base image tags for reproducibility
- Leverage build cache effectively
- Keep images minimal and secure

# **Related Concepts:**
---

**Docker Image Layers**: The foundational concept behind Dockerfile instructions. Each instruction creates a new layer, and understanding layer mechanics is crucial for optimizing Dockerfiles.

**Build Context**: The directory sent to Docker daemon during build. Affects which files are available to COPY/ADD instructions and influences build performance.

**Multi-stage Builds**: Advanced technique using multiple FROM instructions to optimize image size and separate build/runtime environments.

**Layer Caching**: Docker's optimization mechanism that reuses unchanged layers. Understanding caching behavior is essential for efficient Dockerfile design.

**Base Images**: Starting point for all Docker images. Choice of base image affects security, size, and available tools.

**Container Runtime**: The environment where containers execute. Dockerfile instructions like CMD, ENTRYPOINT, and USER affect runtime behavior.

**Union Filesystem**: Underlying technology that combines layers into a single filesystem view. Explains how Dockerfile layers work together.

**Build Arguments (ARG)**: Variables available during build time that can make Dockerfiles more flexible and reusable.

**Parser Directives**: Special comments that control how Docker parses the Dockerfile, enabling advanced features and syntax.

**Security Context**: User permissions, file ownership, and access controls established through USER, COPY --chown, and other instructions.

**Process Management**: How containers handle processes, signals, and lifecycle management affected by ENTRYPOINT and CMD choices.

**Networking**: Container networking concepts related to EXPOSE instruction and port publishing.

**Storage**: Volume management and filesystem concepts related to VOLUME instruction and data persistence.

# **Examples:**
---
## FROM Instruction Examples:

```dockerfile
# FROM - Establishes the base image for the build
# This is the only required instruction in a Dockerfile

# FROM with digest for maximum reproducibility
# Digest is immutable hash of specific image content
FROM ubuntu@sha256:626ffe58f6e7566e00254b638eb7e0f3b11d4da9675088f4781a50ae288f3322

# FROM with multi-architecture support
# --platform specifies target architecture
FROM --platform=linux/amd64 ubuntu:20.04

# Multiple FROM instructions create multi-stage build
# Each FROM starts a new build stage
FROM node:16-alpine AS builder
# ... build instructions ...

FROM nginx:alpine AS production
# ... production setup ...

# FROM with scratch (empty base image)
# Used for minimal images or static binaries
FROM scratch
# Only works with self-contained executables

# FROM with custom registry
# Uses private or alternative registries
FROM myregistry.com/myorg/myimage:v1.0
````

## WORKDIR Instruction Examples:

```dockerfile
# WORKDIR - Sets the working directory for subsequent instructions
# Creates the directory if it doesn't exist

FROM ubuntu:20.04

# Absolute path (preferred)
# All subsequent RUN, CMD, ENTRYPOINT, COPY, ADD use this directory
WORKDIR /usr/src/app

# Subsequent WORKDIR instructions are relative to current WORKDIR
WORKDIR subdir          # Now in /usr/src/app/subdir
WORKDIR ../another      # Now in /usr/src/app/another

# WORKDIR with environment variables
ENV APP_HOME=/opt/myapp
WORKDIR ${APP_HOME}

# Multiple WORKDIR instructions
WORKDIR /tmp
RUN echo "In /tmp directory"
WORKDIR /usr/src/app
RUN echo "In /usr/src/app directory"

# WORKDIR affects where commands run
WORKDIR /usr/src/app
RUN pwd                 # Outputs: /usr/src/app
COPY . .               # Copies files to /usr/src/app
CMD ["./start.sh"]     # Runs from /usr/src/app

# Without WORKDIR, commands run in root directory
# This is less predictable and harder to manage
# RUN cd /usr/src/app && ./start.sh  # Not recommended
```

## COPY and ADD Instruction Examples:

```dockerfile
FROM ubuntu:20.04

# COPY - Copies files/directories from build context to image
# Preferred over ADD for simple file copying

# Basic COPY usage
# Copy single file to current WORKDIR
COPY package.json .

# Copy file to specific destination
COPY package.json /usr/src/app/package.json

# Copy multiple files
COPY package.json package-lock.json ./

# Copy directory and its contents
COPY src/ ./src/

# Copy with wildcard patterns
COPY *.json ./
COPY src/*.js ./src/

# COPY with ownership change
# --chown sets user:group ownership
COPY --chown=appuser:appgroup package.json .

# COPY with permissions
# --chmod sets file permissions (new feature)
COPY --chmod=755 start.sh .

# COPY from previous build stage (multi-stage builds)
FROM node:16-alpine AS builder
RUN npm install && npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html

# ADD - Similar to COPY but with additional features
# Use COPY unless you need ADD's special features

# ADD can extract tar files automatically
ADD app.tar.gz /usr/src/app/

# ADD can fetch files from URLs (not recommended in production)
ADD https://example.com/file.tar.gz /tmp/

# ADD with ownership
ADD --chown=appuser:appgroup app.tar.gz /usr/src/app/

# Best practices for COPY/ADD
WORKDIR /usr/src/app

# Copy package files first for better caching
COPY package*.json ./
RUN npm install

# Copy source code after dependencies
# This allows npm install to be cached if package.json hasn't changed
COPY . .

# Example showing layer caching optimization
FROM node:16-alpine
WORKDIR /app

# This layer will be cached unless package.json changes
COPY package.json .
RUN npm install

# This layer will be rebuilt if any source file changes
COPY src/ ./src/
RUN npm run build
```

## RUN Instruction Examples:

```dockerfile
FROM ubuntu:20.04

# RUN - Executes commands during image build
# Creates new layer with the results

# Basic RUN usage
# Each RUN creates a new layer
RUN apt-get update
RUN apt-get install -y python3

# Combine commands to reduce layers (recommended)
# Use && to chain commands in single RUN instruction
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    curl \
    && rm -rf /var/lib/apt/lists/*

# RUN with line continuation using backslash
RUN apt-get update \
    && apt-get install -y \
        git \
        vim \
        wget \
    && apt-get clean

# RUN with exec form (JSON array)
# Doesn't invoke shell, more efficient
RUN ["apt-get", "update"]
RUN ["apt-get", "install", "-y", "python3"]

# RUN with shell form (default)
# Invokes /bin/sh -c by default
RUN apt-get update && apt-get install -y python3

# RUN with specific shell
RUN ["/bin/bash", "-c", "echo hello"]

# RUN with environment variables
ENV NODE_VERSION=16
RUN curl -fsSL https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-x64.tar.xz | tar -xJ

# RUN with conditional logic
RUN if [ "$NODE_ENV" = "production" ]; then \
        npm ci --only=production; \
    else \
        npm install; \
    fi

# RUN with complex operations
RUN set -ex \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        wget \
    && wget -O /tmp/app.tar.gz "https://example.com/app.tar.gz" \
    && tar -xzf /tmp/app.tar.gz -C /usr/src/app \
    && rm /tmp/app.tar.gz \
    && apt-get purge -y --auto-remove wget \
    && rm -rf /var/lib/apt/lists/*

# RUN for creating users and directories
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
RUN mkdir -p /usr/src/app && chown appuser:appgroup /usr/src/app

# RUN with heredoc (newer Docker versions)
RUN <<EOF
apt-get update
apt-get install -y python3
python3 --version
EOF

# RUN with mount (BuildKit feature)
# --mount=type=cache caches directories between builds
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y python3
```

## CMD and ENTRYPOINT Instruction Examples:

```dockerfile
FROM ubuntu:20.04

# CMD - Sets default command to run when container starts
# Can be overridden by command line arguments

# CMD with exec form (recommended)
# Runs directly without shell
CMD ["python3", "app.py"]

# CMD with shell form
# Runs with /bin/sh -c
CMD python3 app.py

# CMD with parameters to ENTRYPOINT
# When used with ENTRYPOINT, CMD provides default arguments
ENTRYPOINT ["python3", "app.py"]
CMD ["--help"]

# ENTRYPOINT - Sets command that always runs
# Cannot be overridden, only appended to

# ENTRYPOINT with exec form (recommended)
ENTRYPOINT ["python3", "app.py"]

# ENTRYPOINT with shell form
ENTRYPOINT python3 app.py

# Common pattern: ENTRYPOINT + CMD
# ENTRYPOINT defines the executable
# CMD provides default arguments that can be overridden
ENTRYPOINT ["python3", "app.py"]
CMD ["--port", "8080"]

# Example with script entrypoint
COPY docker-entrypoint.sh /
RUN chmod +x /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["app"]

# Practical examples showing differences:

# Example 1: CMD only
FROM nginx:alpine
CMD ["nginx", "-g", "daemon off;"]
# docker run myimage           -> runs nginx
# docker run myimage ls        -> runs ls (overrides CMD)

# Example 2: ENTRYPOINT only
FROM nginx:alpine
ENTRYPOINT ["nginx", "-g", "daemon off;"]
# docker run myimage           -> runs nginx
# docker run myimage ls        -> runs nginx ls (appends to ENTRYPOINT)

# Example 3: ENTRYPOINT + CMD
FROM nginx:alpine
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
# docker run myimage           -> runs nginx -g daemon off;
# docker run myimage -t        -> runs nginx -t (replaces CMD)

# Example 4: Flexible application startup
FROM python:3.9-slim
COPY app.py .
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080", "--host", "0.0.0.0"]
# docker run myimage                           -> python app.py --port 8080 --host 0.0.0.0
# docker run myimage --port 3000              -> python app.py --port 3000
# docker run myimage --debug --port 3000      -> python app.py --debug --port 3000
```

## EXPOSE Instruction Examples:

```dockerfile
FROM ubuntu:20.04

# EXPOSE - Documents which ports the container listens on
# Does NOT publish ports - only for documentation

# Basic EXPOSE usage
# Documents that application listens on port 8080
EXPOSE 8080

# Multiple ports
EXPOSE 8080 9090

# Port with protocol specification
EXPOSE 8080/tcp
EXPOSE 9090/udp

# Port ranges
EXPOSE 8080-8090

# EXPOSE with environment variables
ENV PORT=8080
EXPOSE $PORT

# EXPOSE with multiple protocols
EXPOSE 80/tcp
EXPOSE 443/tcp
EXPOSE 53/udp

# Real-world examples:

# Web application
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]

# Database
FROM postgres:13
EXPOSE 5432
# PostgreSQL default port

# Multi-service application
FROM ubuntu:20.04
# Web server
EXPOSE 80
# API server
EXPOSE 8080
# Admin interface
EXPOSE 9090
# Metrics endpoint
EXPOSE 9100

# Important notes about EXPOSE:
# - EXPOSE is documentation only
# - To actually publish ports, use -p flag with docker run
# - docker run -p 8080:8080 myimage
# - or use docker-compose ports mapping

# EXPOSE in multi-stage build
FROM node:16-alpine AS builder
# ... build steps ...

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

## ENV Instruction Examples:

```dockerfile
FROM ubuntu:20.04

# ENV - Sets environment variables
# Available during build and runtime

# Basic ENV usage
# Single variable
ENV NODE_ENV=production

# Multiple variables in one instruction
ENV NODE_ENV=production \
    PORT=8080 \
    APP_NAME=myapp

# ENV with space-separated format (legacy)
ENV NODE_ENV production

# ENV with complex values
ENV DATABASE_URL="postgresql://user:pass@localhost:5432/mydb"
ENV PATH="/usr/local/app/bin:$PATH"

# ENV referencing other environment variables
ENV APP_HOME=/usr/src/app
ENV LOG_DIR=$APP_HOME/logs

# ENV with default values that can be overridden
ENV PORT=8080
ENV HOST=0.0.0.0
ENV DEBUG=false

# ENV for build-time configuration
ENV BUILD_DEPS="gcc make python3-dev"
RUN apt-get update && apt-get install -y $BUILD_DEPS

# ENV persistence demonstration
ENV TEST_VAR=hello
RUN echo $TEST_VAR          # Outputs: hello
RUN echo ${TEST_VAR}world   # Outputs: helloworld

# ENV vs ARG comparison
# ARG is only available during build
ARG BUILD_VERSION=1.0
ENV APP_VERSION=$BUILD_VERSION

# ENV with conditional logic
ENV NODE_ENV=production
RUN if [ "$NODE_ENV" = "production" ]; then \
        npm ci --only=production; \
    else \
        npm install; \
    fi

# ENV for application configuration
ENV DATABASE_HOST=localhost
ENV DATABASE_PORT=5432
ENV DATABASE_NAME=myapp
ENV REDIS_URL=redis://localhost:6379
ENV LOG_LEVEL=info

# ENV with secrets (not recommended - use secrets management)
# ENV API_KEY=secret123  # DON'T DO THIS

# ENV for runtime behavior
ENV PYTHONPATH=/usr/src/app
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

# ENV in multi-stage builds
FROM node:16-alpine AS builder
ENV NODE_ENV=production
ENV BUILD_TARGET=production
RUN npm run build

FROM nginx:alpine
ENV NGINX_WORKER_PROCESSES=auto
ENV NGINX_WORKER_CONNECTIONS=1024
```

## VOLUME Instruction Examples:

```dockerfile
FROM ubuntu:20.04

# VOLUME - Creates mount point for external storage
# Marks directories as externally mountable

# Basic VOLUME usage
# Single volume
VOLUME /data

# Multiple volumes
VOLUME /data /logs /config

# VOLUME with JSON array format
VOLUME ["/data", "/logs"]

# VOLUME with application-specific directories
VOLUME /var/lib/mysql        # Database data
VOLUME /var/log/nginx        # Log files
VOLUME /etc/nginx/conf.d     # Configuration files

# VOLUME with environment variables
ENV DATA_DIR=/usr/src/app/data
VOLUME $DATA_DIR

# Database example
FROM postgres:13
# PostgreSQL data directory
VOLUME /var/lib/postgresql/data
ENV POSTGRES_DB=mydb
ENV POSTGRES_USER=user
ENV POSTGRES_PASSWORD=password

# Web application example
FROM nginx:alpine
# Static content volume
VOLUME /usr/share/nginx/html
# Configuration volume
VOLUME /etc/nginx/conf.d
# Log volume
VOLUME /var/log/nginx

# Application with multiple volumes
FROM node:16-alpine
WORKDIR /app

# Application code (typically not a volume in production)
COPY package*.json ./
RUN npm install
COPY . .

# Data persistence
VOLUME /app/data
# Log files
VOLUME /app/logs
# Upload directory
VOLUME /app/uploads

EXPOSE 3000
CMD ["npm", "start"]

# Volume considerations and best practices:

# 1. Volumes are created automatically if they don't exist
# 2. Data in volumes persists after container removal
# 3. Volumes can be shared between containers
# 4. Changes to volume after VOLUME instruction are discarded

# Example showing volume behavior
FROM ubuntu:20.04
RUN mkdir /data
RUN echo "initial data" > /data/file.txt
VOLUME /data
# Any changes to /data after this point are lost
RUN echo "this will be lost" > /data/lost.txt

# Named volumes vs bind mounts
# Named volume (managed by Docker)
# docker run -v myvolume:/data myimage

# Bind mount (host directory)
# docker run -v /host/path:/data myimage

# Anonymous volume
# docker run -v /data myimage

# Volume with initialization script
FROM mysql:8.0
VOLUME /var/lib/mysql
COPY init.sql /docker-entrypoint-initdb.d/
# Scripts in /docker-entrypoint-initdb.d/ run on first startup
```

## USER Instruction Examples:

```dockerfile
FROM ubuntu:20.04

# USER - Sets user context for subsequent instructions
# Affects RUN, CMD, ENTRYPOINT

# Default user is root (UID 0)
# Not recommended for security reasons

# Create user first, then switch
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser

# USER with UID (numeric)
USER 1000

# USER with UID and GID
USER 1000:1000

# USER with username and group
USER appuser:appgroup

# Switching back to root (if needed)
USER root
RUN apt-get update && apt-get install -y some-package
USER appuser

# Node.js application example
FROM node:16-alpine

# Create app directory
RUN mkdir -p /usr/src/app

# Create user for security
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# Set up application
WORKDIR /usr/src/app

# Install dependencies as root
COPY package*.json ./
RUN npm ci --only=production

# Copy application code and set ownership
COPY --chown=nextjs:nodejs . .

# Switch to non-root user
USER nextjs

# Application runs as non-root user
EXPOSE 3000
CMD ["node", "server.js"]

# Python application example
FROM python:3.9-slim

# Create non-root user
RUN useradd --create-home --shell /bin/bash app

# Set working directory
WORKDIR /home/app

# Install dependencies as root
COPY requirements.txt .
RUN pip install -r requirements.txt

# Copy application and set ownership
COPY --chown=app:app . .

# Switch to non-root user
USER app

# Run application
CMD ["python", "app.py"]

# Multi-stage build with USER
FROM python:3.9-slim AS builder
# Build stage runs as root
RUN pip install build-tools
COPY . .
RUN python setup.py build

FROM python:3.9-slim AS runtime
# Create user for runtime
RUN useradd --create-home --shell /bin/bash app
USER app
WORKDIR /home/app

# Copy built application
COPY --from=builder --chown=app:app /app/dist ./dist
CMD ["python", "-m", "dist"]

# Security considerations:

# 1. Never run production containers as root
# 2. Use specific UID/GID numbers for consistency
# 3. Ensure user has minimal required permissions
# 4. Use --chown flag with COPY to set ownership

# File permissions example
FROM ubuntu:20.04

# Create user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# Create directories with proper ownership
RUN mkdir -p /usr/src/app /var/log/app
RUN chown -R appuser:appgroup /usr/src/app /var/log/app

# Copy files with ownership
COPY --chown=appuser:appgroup . /usr/src/app/

# Make script executable
RUN chmod +x /usr/src/app/start.sh

# Switch to non-root user
USER appuser

# User context affects runtime
WORKDIR /usr/src/app
CMD ["./start.sh"]
```

## Advanced Syntax Examples:

```dockerfile
# Parser directive (must be first line)
# syntax=docker/dockerfile:1.4

FROM ubuntu:20.04

# Here documents (heredoc) - BuildKit feature
RUN <<EOF
apt-get update
apt-get install -y curl
curl --version
EOF

# Multi-line strings with heredoc
RUN <<'EOF'
cat > /etc/nginx/nginx.conf << 'NGINX_CONF'
user nginx;
worker_processes auto;
events {
    worker_connections 1024;
}
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    sendfile on;
    keepalive_timeout 65;
    
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
}
NGINX_CONF
EOF

# Conditional builds with BuildKit
FROM ubuntu:20.04 AS base
RUN apt-get update

FROM base AS development
RUN apt-get install -y debug-tools

FROM base AS production
RUN apt-get install -y --no-install-recommends minimal-tools

# Build secrets (don't persist in image)
RUN --mount=type=secret,id=api_key \
    API_KEY=$(cat /run/secrets/api_key) && \
    curl -H "Authorization: Bearer $API_KEY" https://api.example.com/setup

# Cache mounts for package managers
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt \
    apt-get update && apt-get install -y python3

# SSH mounts for private repositories
RUN --mount=type=ssh \
    git clone git@github.com:private/repo.git /app

# Build context mounts
RUN --mount=type=bind,source=.,target=/src \
    cp /src/config.json /app/config.json

# Complex multi-stage with build arguments
ARG NODE_VERSION=16
ARG ENVIRONMENT=production

FROM node:${NODE_VERSION}-alpine AS base
WORKDIR /app

FROM base AS dependencies
COPY package*.json ./
RUN npm ci

FROM dependencies AS build
COPY . .
RUN npm run build

FROM base AS development
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
CMD ["npm", "run", "dev"]

FROM nginx:alpine AS production
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

# **Flashcards:**
---

What is the difference between COPY and ADD instructions in Docker?;; COPY simply copies files/directories from build context to image filesystem. ADD has additional features like automatic tar extraction and URL fetching, but COPY is preferred for simple file operations as it's more predictable and explicit.

What is the difference between CMD and ENTRYPOINT instructions?;; CMD sets the default command that can be overridden by docker run arguments. ENTRYPOINT sets the command that always runs and cannot be overridden - docker run arguments are appended to it. They're often used together where ENTRYPOINT defines the executable and CMD provides default arguments.

Why should you use the exec form ["cmd", "arg"] instead of shell form for CMD and ENTRYPOINT?;; Exec form runs the command directly without invoking a shell, making it more efficient and ensuring proper signal handling. Shell form runs commands through /bin/sh -c, which can interfere with signal propagation and process management.

What is the purpose of the EXPOSE instruction and what doesn't it do?;; EXPOSE documents which ports the container listens on for communication between containers and documentation purposes. It does NOT publish ports to the host - you must use -p flag with docker run or ports mapping in docker-compose to actually publish ports.

How does the USER instruction affect container security?;; USER sets the user context for subsequent instructions and container runtime. Running containers as non-root users (instead of default root) is a security best practice that limits potential damage from container compromise and follows the principle of least privilege.

What happens to filesystem changes made after a VOLUME instruction?;; Any changes made to the volume directory after the VOLUME instruction are discarded during the build process. The VOLUME instruction marks the directory as a mount point, and Docker ignores subsequent changes to that path in the Dockerfile.