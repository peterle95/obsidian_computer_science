---
memory: to_finish
tags:
  - will_learn
language:
  - Docker
review-date:
last-reviewed: ""
scheda: done
visit-count: 0
confidence-level: 1
consecutive-correct: 0
last-struggle-date: ""
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

# **Purpose/Why**:
---

The Docker Compose syntax provides a declarative, human-readable way to define and configure all the components of a multi-container application in a single file (`docker-compose.yml`). The fundamental problem it solves is the complexity of managing an application's architecture, including its services, networking, and data persistence. Manually running and connecting multiple containers is tedious and error-prone.

Its primary application is to serve as a blueprint for creating reproducible application environments. By codifying the application's entire stack—from which Docker images to use or how to build them, to how containers should communicate and store data—the YAML syntax ensures consistency across different stages of the development lifecycle (development, testing, production). This is critically important for eliminating "it works on my machine" problems and enabling seamless collaboration among team members. The clear and structured syntax makes complex application setups understandable and manageable.

# **Core Explanation:**
---

The Docker Compose syntax is a set of keys and values defined in a YAML file, typically `docker-compose.yml`, that describes a multi-container application. This file is the central piece that the `docker compose` command-line tool uses to manage the application's lifecycle.

**Key Syntax Elements:**

*   **`version`**: (Legacy) This top-level key specifies the version of the Docker Compose file format. While modern versions of Docker Compose are largely backward-compatible and the `version` tag is often optional, it was historically used to determine which features were available. For example, `version: '3.8'` was a common declaration.

*   **`services`**: This is a required top-level key that defines all the individual, isolated components of your application. Each key under `services` is the name of a service (e.g., `web`, `db`, `api`) and its value contains the configuration for the container(s) running that service.

*   **`image`**: Used within a service definition, this key specifies the Docker image to use for creating the container. This can be an image from a public registry like Docker Hub (e.g., `nginx:latest`) or a private registry.

*   **`build`**: This key is used when you need to build a custom image from a Dockerfile instead of pulling a pre-existing one. Its value is typically the path to the directory containing the Dockerfile (e.g., `build: .`).

*   **`ports`**: This key maps ports between the host machine and the container in the format `"HOST_PORT:CONTAINER_PORT"`. This allows you to access a service running inside a container from your local machine. For example, `ports: - "8080:80"` maps port 8080 on the host to port 80 in the container.

*   **`volumes`**: This key is used for persisting data generated by and used by containers. It can be used in two ways:
    1.  **Named Volumes**: Declared under a top-level `volumes` key and then attached to a service. This is the preferred way to manage persistent data.
    2.  **Bind Mounts**: Maps a host directory directly into a container (e.g., `./app:/code`), which is useful for development when you want code changes on the host to be reflected immediately in the container.

*   **`environment`**: This key allows you to set environment variables inside a container. These are often used for configuration, such as setting database credentials, API keys, or enabling debug modes. They can be provided as a list or a map.

*   **`depends_on`**: This key defines dependencies between services, controlling the startup and shutdown order. If service `web` depends on service `db`, Docker Compose will start `db` before starting `web`. It's important to note this does not wait for the dependent service to be "ready," only for it to have started.

*   **`networks`**: While Docker Compose creates a default network for all services in the file, this key allows you to define more complex network topologies. You can create custom networks under the top-level `networks` key and then assign services to specific networks.

# **Related Concepts:**
---

*   **YAML (YAML Ain't Markup Language):** Docker Compose files are written in YAML, a human-readable data serialization standard. Its indentation-based structure is what gives the `docker-compose.yml` file its clarity and organization. Understanding basic YAML syntax (key-value pairs, lists, and indentation) is essential for writing Compose files.

*   **Dockerfile:** A Dockerfile is a script for building a *single* Docker image. The `build` key in a Docker Compose service definition directly references a Dockerfile. While a Dockerfile defines the *contents* of an image, the `docker-compose.yml` file defines how one or more images (and the containers created from them) should be *run and orchestrated* together.

*   **Container Networking:** The `networks` key in Docker Compose is a high-level abstraction over Docker's networking features. By default, Compose creates a bridge network that allows services to discover and communicate with each other using their service names as hostnames. The syntax allows you to customize this behavior, for instance, by creating multiple isolated networks.

*   **Data Persistence:** The `volumes` key is directly related to the concept of managing persistent data in ephemeral containers. Docker containers are stateless by default; any data written inside a container is lost when the container is removed. Volumes provide a mechanism to store data outside the container's lifecycle, ensuring it persists.

# **Examples:**
---

```yaml
# Specifies the file format version. Modern Docker Compose often doesn't require this,
# but it's good practice for compatibility.
version: '3.8'

# Top-level key that defines all the application's services.
services:
  # Defines a service named 'webapp'.
  webapp:
    # Instructs Compose to build an image from the Dockerfile in the current directory.
    build: .
    # Maps port 8000 on the host machine to port 5000 inside the container.
    ports:
      - "8000:5000"
    # Mounts the current directory on the host to the /app directory in the container.
    # This is a bind mount, useful for development.
    volumes:
      - .:/app
    # Sets environment variables for the webapp container.
    environment:
      - DATABASE_URL=postgres://user:password@db:5432/mydatabase
      - FLASK_ENV=development
    # Specifies that the 'webapp' service depends on the 'db' and 'redis' services.
    # 'db' and 'redis' will be started before 'webapp'.
    depends_on:
      - db
      - redis
    # Connects this service to the 'frontend' and 'backend' networks.
    networks:
      - frontend
      - backend

  # Defines a service named 'db' for the PostgreSQL database.
  db:
    # Pulls the specified PostgreSQL image from Docker Hub.
    image: postgres:13-alpine
    # Sets environment variables required to initialize the PostgreSQL container.
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydatabase
    # Mounts a named volume 'db_data' to the directory where PostgreSQL stores its data.
    # This ensures that database data persists even if the container is removed.
    volumes:
      - db_data:/var/lib/postgresql/data
    # Connects this service only to the 'backend' network for security.
    networks:
      - backend

  # Defines a service named 'redis' for caching.
  redis:
    # Pulls the official Redis image.
    image: "redis:alpine"
    # Connects this service only to the 'backend' network.
    networks:
      - backend

# Top-level key to declare named volumes.
volumes:
  # Declares a named volume called 'db_data'. Docker manages this volume.
  db_data:

# Top-level key to define custom networks.
networks:
  # Defines a network named 'frontend'.
  frontend:
  # Defines a network named 'backend'.
  backend:
```

# **Flashcards:**
---

What is the purpose of the `image` key in a Docker Compose service definition?;;It specifies the Docker image to use for creating the service's container.
What is the difference between the `image` and `build` keys in Docker Compose?;;`image` pulls a pre-existing image from a registry, while `build` creates a new image from a local Dockerfile.
How do you map a port from the host to a container using Docker Compose syntax?;;Use the `ports` key with the format `"HOST_PORT:CONTAINER_PORT"`.
What does the `depends_on` key control in a `docker-compose.yml` file?;;It controls the startup and shutdown order of services, ensuring dependencies are started first.
What are the two main ways to use the `volumes` key for data persistence?;;1. Using a named volume declared at the top level for Docker-managed persistence. 2. Using a bind mount to map a host directory directly into a container.
How can you define and use a custom network for specific services in Docker Compose?;;Define the network under the top-level `networks` key, and then assign services to it using the `networks` key within the service definition.