---
memory: to_finish
tags:
 - learned
language:
 - Docker
review-date: ""
last-reviewed: 2025-08-17
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 3
last-struggle-date: 2025-07-18
cssclasses:

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

# **Purpose/Why**:

---
Docker Compose solves the complexity of managing ==multi-container applications by providing a declarative way to define, configure, and orchestrate multiple Docker containers as a single application stack.== Instead of manually running multiple `docker run` commands with complex flags and managing container dependencies, networking, and volumes separately, docker-compose.yml allows you to define your entire application architecture in a single YAML file.

This is crucial for modern application development because most real-world applications consist of multiple services (web server, database, cache, message queue, etc.) that need to work together. Docker Compose eliminates the manual coordination overhead, ensures consistent environments across development/staging/production, and makes it easy to share and reproduce complex application setups.

# **Core Explanation:**

---
A docker-compose.yml file is a YAML configuration file that defines a multi-container Docker application. It uses a declarative syntax to specify:

**Key Components:**

- **Services**: Individual containers that make up your application
- **Networks**: Custom networks for inter-container communication
- **Volumes**: Persistent data storage and sharing between containers
- **Environment Variables**: Configuration values passed to containers
- **Dependencies**: Startup order and service relationships

**Structure and Syntax:**

- Uses YAML format with specific Docker Compose schema
- Organizes configuration into logical sections (services, networks, volumes)
- Supports variable substitution and extends functionality
- Version-controlled alongside your application code

**How it Works:**

1. Define your application stack in docker-compose.yml
2. Run `docker-compose up` to create and start all services
3. Docker Compose creates a shared network for service communication
4. Services can reference each other by name for networking
5. Volumes persist data beyond container lifecycle
6. Use `docker-compose down` to stop and remove all resources

**Key Characteristics:**

- **Declarative**: Describe desired state, not steps to achieve it
- **Reproducible**: Same configuration works across different environments
- **Scalable**: Easy to scale services up or down
- **Portable**: Share entire application stack as a single file

# **Related Concepts:**

---
**Docker Containers**: The fundamental unit that Compose orchestrates. Each service in docker-compose.yml becomes one or more containers.

**Docker Images**: The templates used to create containers. Services reference images either from registries or built from Dockerfiles.

**Docker Networks**: Enable communication between containers. Compose automatically creates networks but you can define custom ones.

**Docker Volumes**: Provide persistent storage. Compose can create named volumes or bind mounts for data persistence.

**Kubernetes**: A more complex orchestration platform. While Compose is great for development and simple deployments, Kubernetes handles production-scale orchestration with advanced features like auto-scaling and health checks.

**Docker Swarm**: Docker's native clustering solution. Compose files can be used with Swarm for production deployments.

**Environment Files (.env)**: Work with Compose to externalize configuration values, making applications more portable.

**Dockerfile**: Used to build custom images that can be referenced in Compose services.

# **Examples:**

---
```yaml

# docker-compose.yml - Complete web application stack
version: '3.8'

# Specifies Docker Compose file format version

services:

# Web application service
 web:
 build: .

# Build image from Dockerfile in current directory
 ports:
 - "3000:3000"

# Map host port 3000 to container port 3000
 depends_on:
 - db

# Ensure database starts before web service
 - redis

# Ensure Redis starts before web service
 environment:
 - NODE_ENV=development

# Set environment variable
 - DB_HOST=db

# Reference database service by name
 - REDIS_URL=redis://redis:6379

# Reference Redis service
 volumes:
 - .:/app

# Mount current directory to /app in container (for development)
 - node_modules:/app/node_modules

# Use named volume for node_modules
 networks:
 - app-network

# Connect to custom network

# Database service
 db:
 image: postgres:13

# Use official PostgreSQL image
 environment:
 - POSTGRES_DB=myapp

# Create database named 'myapp'
 - POSTGRES_USER=user

# Set database username
 - POSTGRES_PASSWORD=password

# Set database password
 volumes:
 - postgres_data:/var/lib/postgresql/data

# Persist database data
 - ./init.sql:/docker-entrypoint-initdb.d/init.sql

# Initialize database with SQL script
 networks:
 - app-network

# Connect to same network as web service

# Redis cache service
 redis:
 image: redis:6-alpine

# Use lightweight Redis image
 ports:
 - "6379:6379"

# Expose Redis port (optional for external access)
 volumes:
 - redis_data:/data

# Persist Redis data
 networks:
 - app-network

# Background worker service
 worker:
 build: .

# Same image as web service
 command: npm run worker

# Override default command to run worker
 depends_on:
 - db
 - redis
 environment:
 - NODE_ENV=development
 - DB_HOST=db
 - REDIS_URL=redis://redis:6379
 volumes:
 - .:/app

# Same volume mount as web service
 networks:
 - app-network

# Define named volumes for data persistence
volumes:
 postgres_data:

# Stores database data
 redis_data:

# Stores Redis data
 node_modules:

# Stores Node.js dependencies

# Define custom networks
networks:
 app-network:

# Custom network for service communication
 driver: bridge

# Use bridge network driver
```

```yaml

# docker-compose.override.yml - Development-specific overrides
version: '3.8'

services:
 web:

# Override for development - enable hot reloading
 volumes:
 - .:/app

# Mount source code for live updates
 environment:
 - DEBUG=true

# Enable debug mode
 command: npm run dev

# Use development server

 db:
 ports:
 - "5432:5432"

# Expose database port for local access
```

```yaml

# docker-compose.prod.yml - Production configuration
version: '3.8'

services:
 web:

# Production optimizations
 restart: unless-stopped

# Auto-restart on failure
 environment:
 - NODE_ENV=production

# Remove volume mounts - use built image as-is

 db:
 restart: unless-stopped

# No port exposure - only internal access
 environment:
 - POSTGRES_PASSWORD_FILE=/run/secrets/db_password

# Use secrets
 secrets:
 - db_password

secrets:
 db_password:
 file: ./db_password.txt

# External password file
```

# **Flashcards:**

---
What is the primary purpose of a docker-compose.yml file?;; To define and orchestrate multi-container Docker applications in a single declarative YAML file, managing services, networks, volumes, and dependencies together.

What are the four main sections of a docker-compose.yml file?;; Services (individual containers), Networks (communication between containers), Volumes (persistent storage), and Environment variables (configuration values).

How do containers communicate with each other in Docker Compose?;; Containers can reference each other by service name within the same network. Docker Compose automatically creates a default network and provides DNS resolution between services.

What is the difference between `build` and `image` in a service definition?;; `build` creates a custom image from a Dockerfile in the specified directory, while `image` uses an existing image from a registry like Docker Hub.

What does the `depends_on` directive do in Docker Compose?;; It defines startup dependencies between services, ensuring that dependent services start before the current service, though it doesn't wait for the service to be "ready" - only started.

How do you persist data in Docker Compose?;; Use volumes - either named volumes defined in the volumes section for Docker-managed storage, or bind mounts to map host directories to container paths.