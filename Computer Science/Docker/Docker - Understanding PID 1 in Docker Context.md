---
memory: to_finish
tags:
  - will_learn
language:
  - Docker
review-date:
last-reviewed:
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

PID 1 understanding solves the fundamental problem of **proper process lifecycle management in containerized environments**. 

In Docker containers, the PID 1 process has special responsibilities that differ from regular processes:
- **Signal propagation**: It must correctly handle and forward termination signals (SIGTERM, SIGINT)
- **Zombie reaping**: It must clean up orphaned child processes that would otherwise accumulate as zombies
- **Graceful shutdown**: It enables proper container shutdown sequences when Docker sends stop commands

**Why it matters**:
- Improper PID 1 handling leads to containers that won't stop gracefully, requiring force kills (SIGKILL)
- Services running as child processes (not PID 1) may not receive shutdown signals at all
- Accumulated zombie processes waste system resources
- Production containers require reliable start/stop behavior for orchestration (Kubernetes, Docker Swarm)

This is critical in system administration and DevOps because containers are designed to run **one primary process**, not emulate full virtual machines. Understanding PID 1 is the difference between amateur "it works on my machine" containers and production-ready containerized services.

# **Core Explanation:**
---

## What is PID 1?

**PID 1** (Process ID 1) is the first process started in any Unix-like system. In traditional operating systems, this is the init system (systemd, SysVinit). In Docker containers, **whatever command you specify as ENTRYPOINT/CMD becomes PID 1**.

## Key Characteristics

### 1. Signal Handling Responsibility
- PID 1 has **different signal handling** than other processes
- By default, the kernel won't terminate PID 1 on signals unless the process explicitly handles them
- Docker sends **SIGTERM** when stopping a container (with a 10-second grace period before SIGKILL)
- If PID 1 doesn't handle SIGTERM, the container won't shut down gracefully

### 2. Zombie Process Reaping
- When a child process dies, it becomes a "zombie" until its parent calls `wait()` to collect its exit status
- PID 1 **must reap zombie processes** for any orphaned children
- If your script spawns children without proper cleanup, zombies accumulate

### 3. Propagation Issues
- If PID 1 is a shell script that starts services in the background, those services don't receive signals directly
- Example: `bash -c "nginx && tail -f /dev/null"` makes bash PID 1, not nginx

## How Docker Containers Differ from VMs

Containers are **NOT virtual machines**:
- VMs run a full OS with init system managing multiple services
- Containers run **one primary application** as PID 1
- No init system is needed (unless you explicitly add one like tini)
- [[Docker - Docker vs. Virtual Machines]]

## The "Hacky Patches" Problem

Commands like these are anti-patterns:
```bash
tail -f /dev/null        # Blocks forever but doesn't manage anything
sleep infinity           # Same issue
while true; do sleep 1; done  # Infinite loop doing nothing
bash                     # Interactive shell that blocks
```

**Why they're bad**:
- They become PID 1 but don't actually run your service
- Your real service (nginx, MariaDB) runs as a **child process**
- Signals don't reach your service properly
- Container doesn't restart correctly
- Violates Docker's design philosophy

## Proper Approach: Run Service as PID 1

### Daemon Mode OFF
Most services have a daemon flag that forks to background. **Turn this off**:
- `nginx -g "daemon off;"` - runs in foreground
- `php-fpm -F` - foreground mode (not `-D` daemon mode)
- `mysqld` - already runs in foreground by default

### Exec Form vs Shell Form
```dockerfile
# EXEC FORM (recommended) - direct process execution
CMD ["nginx", "-g", "daemon off;"]
# Process tree: nginx (PID 1)

# SHELL FORM (avoid) - wraps in /bin/sh -c
CMD nginx -g "daemon off;"
# Process tree: /bin/sh (PID 1) → nginx (child)
```

### Using exec in Scripts

If you need initialization before starting your service:
```bash
#!/bin/bash
# Do setup work
initialize_database

# CRITICAL: use 'exec' to replace this shell with the service
exec mysqld
# After this line, mysqld becomes PID 1, bash is gone
```

The `exec` command **replaces** the current process instead of spawning a child.

## Best Practices Summary

1. ✅ Run your main service directly as PID 1
2. ✅ Use exec form in Dockerfile (`CMD ["executable", "param"]`)
3. ✅ Disable daemon modes (run in foreground)
4. ✅ Use `exec` in entrypoint scripts to replace the shell
5. ✅ Handle signals properly (or use a minimal init like `tini`)
6. ❌ Never use blocking no-op commands (`tail -f`, `sleep infinity`)
7. ❌ Don't run services in background with shell keeping container alive

# **Related Concepts:**
---
## 1. **Init Systems (systemd, SysVinit, runit)**
Traditional Linux init systems that act as PID 1 in full operating systems. They:
- Manage multiple services
- Handle dependencies between services
- Provide service supervision and restart

**Connection**: Docker containers intentionally avoid these because they're designed for single-process models. However, you can use lightweight inits like `tini` or `dumb-init` if needed.

## 2. **Process Signals (SIGTERM, SIGKILL, SIGINT)**
Unix signals for inter-process communication:
- `SIGTERM` (15): Polite "please terminate" - allows cleanup
- `SIGKILL` (9): Forceful kill - cannot be caught or ignored
- `SIGINT` (2): Interrupt from keyboard (Ctrl+C)

**Connection**: Docker sends SIGTERM to PID 1 when stopping. If not handled in 10 seconds (configurable), Docker sends SIGKILL.

## 3. **Zombie Processes**
Processes that have completed execution but still have an entry in the process table:
- Appear as `<defunct>` in `ps` output
- Consume minimal resources but accumulate
- Must be reaped by parent process calling `wait()`

**Connection**: PID 1 must reap zombies for any orphaned children, or they accumulate forever.

## 4. **Docker ENTRYPOINT vs CMD**
Two Dockerfile instructions for defining container startup:
- `ENTRYPOINT`: Main executable (harder to override)
- `CMD`: Default arguments (easily overridden)
- Combined: `ENTRYPOINT` is executable, `CMD` provides default args

**Connection**: Both determine what becomes PID 1. Best practice is ENTRYPOINT for executable, CMD for configurable parameters.

## 5. **exec vs fork System Calls**
Unix process creation primitives:
- `fork()`: Creates child process (copy of parent)
- `exec()`: Replaces current process with new program

**Connection**: Using `exec` in shell scripts replaces the shell with your service, making it PID 1. Without `exec`, your service is a child of the shell.

## 6. **Container Restart Policies**
Docker's automatic restart configurations:
- `no`: Never restart (default)
- `always`: Always restart on crash
- `unless-stopped`: Restart unless manually stopped
- `on-failure`: Restart only on non-zero exit

**Connection**: Proper PID 1 handling ensures restart policies work correctly. If PID 1 doesn't exit properly, restart logic fails.

## 7. **Process Supervision (supervisord, s6)**
Tools that monitor and restart processes:
- Used in traditional servers to keep services running
- Handle multiple processes in one container (anti-pattern in Docker)

**Connection**: In Docker, process supervision should be handled by Docker/Kubernetes, not within containers. Each service should be its own container.

## 8. **Graceful Shutdown**
Proper application termination that:
- Completes in-flight requests
- Closes database connections
- Flushes buffers
- Saves state

**Connection**: Only possible if PID 1 receives and handles SIGTERM correctly.

## 9. **Tini and dumb-init**
Minimal init systems designed for containers:
- Handle signal forwarding
- Reap zombie processes
- Extremely lightweight (~10KB)

**Connection**: Use these if your application doesn't handle signals well or spawns many child processes. They become PID 1 and manage your actual service.

# **Examples:**
---
## Example 1: Bad Practice - Shell Wrapper Keeps Container Alive
```dockerfile
# ❌ BAD: This Dockerfile violates Docker best practices
FROM debian:11

RUN apt-get update && apt-get install -y nginx

# PROBLEM: Shell form wraps command in /bin/sh
# Process tree will be: /bin/sh (PID 1) → nginx (child)
# When Docker sends SIGTERM, it goes to /bin/sh, not nginx
CMD nginx -g "daemon off;" && tail -f /dev/null

# RESULT:
# - /bin/sh becomes PID 1
# - nginx runs as background child
# - tail -f keeps container alive but serves no purpose
# - SIGTERM to PID 1 doesn't reach nginx
# - Container takes 10 seconds to stop (grace period) then gets SIGKILL
```

## Example 2: Good Practice - Service Directly as PID 1
```dockerfile
# ✅ GOOD: Nginx runs directly as PID 1
FROM debian:11

RUN apt-get update && apt-get install -y nginx

# Exec form: nginx runs directly, no shell wrapper
# "daemon off" tells nginx to stay in foreground (not fork to background)
CMD ["nginx", "-g", "daemon off;"]

# RESULT:
# - nginx is PID 1
# - Receives SIGTERM directly from Docker
# - Can shut down gracefully (finish requests, close connections)
# - Container stops in <1 second typically
```

## Example 3: MariaDB with Initialization Script (Proper exec Usage)
```dockerfile
# ✅ GOOD: MariaDB with initialization, using exec properly
FROM debian:11

RUN apt-get update && \
    apt-get install -y mariadb-server && \
    rm -rf /var/lib/apt/lists/*

# Copy initialization script
COPY init-mariadb.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/init-mariadb.sh

EXPOSE 3306

# Entrypoint runs our script, which will exec into mysqld
ENTRYPOINT ["/usr/local/bin/init-mariadb.sh"]
```
```bash
#!/bin/bash
# init-mariadb.sh
# This script runs as PID 1 initially, but exec replaces it with mysqld

set -e  # Exit on error

echo "Initializing MariaDB..."

# Check if database is already initialized
if [ ! -d "/var/lib/mysql/mysql" ]; then
    echo "First run - installing database..."
    
    # Install system tables
    mysql_install_db --user=mysql --datadir=/var/lib/mysql
    
    # Start mysqld temporarily in background to run setup commands
    mysqld --user=mysql --datadir=/var/lib/mysql &
    MYSQL_PID=$!
    
    # Wait for MySQL to be ready
    until mysqladmin ping &>/dev/null; do
        echo "Waiting for MariaDB to start..."
        sleep 1
    done
    
    # Run initialization SQL
    mysql -u root <<-EOSQL
        CREATE DATABASE IF NOT EXISTS ${DB_NAME};
        CREATE USER IF NOT EXISTS '${DB_USER}'@'%' IDENTIFIED BY '${DB_PASSWORD}';
        GRANT ALL PRIVILEGES ON ${DB_NAME}.* TO '${DB_USER}'@'%';
        FLUSH PRIVILEGES;
EOSQL
    
    # Stop the temporary instance
    mysqladmin -u root shutdown
    wait $MYSQL_PID
    
    echo "Database initialized successfully"
fi

echo "Starting MariaDB as PID 1..."

# CRITICAL: exec replaces this bash script with mysqld
# After this line executes:
# - bash process (current PID 1) is gone
# - mysqld becomes PID 1
# - mysqld receives all signals directly
exec mysqld --user=mysql --datadir=/var/lib/mysql

# This line never executes - process has been replaced
```

## Example 4: PHP-FPM Container (Comparing Bad vs Good)
```dockerfile
# ❌ BAD VERSION
FROM debian:11
RUN apt-get update && apt-get install -y php-fpm

# Bad: php-fpm runs in background (-D flag), sleep keeps container alive
CMD ["sh", "-c", "php-fpm -D && sleep infinity"]

# Process tree:
# sh (PID 1)
#   ├── php-fpm master (background daemon)
#   └── sleep infinity (blocking)
# SIGTERM goes to sh, not php-fpm master
```
```dockerfile
# ✅ GOOD VERSION
FROM debian:11
RUN apt-get update && apt-get install -y php-fpm

# Configure php-fpm to not daemonize
RUN sed -i 's/;daemonize = yes/daemonize = no/' /etc/php/7.4/fpm/php-fpm.conf

# Good: -F flag runs php-fpm in foreground as PID 1
CMD ["php-fpm", "-F"]

# Process tree:
# php-fpm (PID 1)
#   ├── php-fpm pool process
#   └── php-fpm pool process
# SIGTERM goes directly to php-fpm master
# Graceful shutdown: finishes processing requests, then exits
```

## Example 5: Multi-Command Initialization with Proper Exec
```bash
#!/bin/bash
# entrypoint.sh for WordPress setup

set -e  # Exit immediately if any command fails

# Function to wait for database to be ready
wait_for_db() {
    echo "Waiting for database at ${DB_HOST}..."
    
    # Keep trying to connect until successful
    # This loop is OK because it's not keeping container alive - it's setup
    until mysql -h"${DB_HOST}" -u"${DB_USER}" -p"${DB_PASSWORD}" -e "SELECT 1" &>/dev/null; do
        echo "Database not ready yet..."
        sleep 2
    done
    
    echo "Database is ready!"
}

# Wait for MariaDB container to be ready
wait_for_db

# Download WordPress if not present
if [ ! -f /var/www/html/wp-config.php ]; then
    echo "Installing WordPress..."
    
    # Download and extract WordPress
    cd /var/www/html
    wp core download --allow-root
    
    # Configure WordPress
    wp config create \
        --dbname="${DB_NAME}" \
        --dbuser="${DB_USER}" \
        --dbpass="${DB_PASSWORD}" \
        --dbhost="${DB_HOST}" \
        --allow-root
    
    # Install WordPress
    wp core install \
        --url="${DOMAIN_NAME}" \
        --title="${WP_TITLE}" \
        --admin_user="${WP_ADMIN_USER}" \
        --admin_password="${WP_ADMIN_PASSWORD}" \
        --admin_email="${WP_ADMIN_EMAIL}" \
        --allow-root
    
    echo "WordPress installed successfully"
fi

# Set proper permissions
chown -R www-data:www-data /var/www/html

echo "Starting PHP-FPM as PID 1..."

# CRITICAL: exec replaces bash with php-fpm
# php-fpm now becomes PID 1 and receives all signals
exec php-fpm -F

# Everything after exec is unreachable
```

## Example 6: Using Tini as Minimal Init (Alternative Approach)
```dockerfile
# ✅ ALTERNATIVE: Using tini when your app doesn't handle signals well
FROM debian:11

# Install tini - a minimal init system for containers
RUN apt-get update && \
    apt-get install -y tini nginx && \
    rm -rf /var/lib/apt/lists/*

# tini becomes PID 1
ENTRYPOINT ["/usr/bin/tini", "--"]

# nginx runs as child of tini
# tini forwards signals properly and reaps zombies
CMD ["nginx", "-g", "daemon off;"]

# Process tree:
# tini (PID 1)
#   └── nginx (child)
# 
# Benefits:
# - tini handles signal forwarding to nginx
# - tini reaps any zombie processes
# - Useful if your application spawns multiple children
```

## Example 7: Checking PID 1 in Running Container
```bash
# Start a container with bad PID 1 handling
docker run -d --name bad_container nginx-bad:latest

# Check what process is PID 1
docker exec bad_container ps aux

# Output might show:
# USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
# root         1  0.0  0.1  18504  3384 ?        Ss   10:00   0:00 /bin/sh -c nginx && tail -f /dev/null
# root         7  0.0  0.5  32456  5672 ?        S    10:00   0:00 nginx: master process
# 
# PROBLEM: PID 1 is /bin/sh, not nginx

# Try to stop gracefully
docker stop bad_container
# Takes full 10 seconds (grace period) before forceful kill

# Compare with good container
docker run -d --name good_container nginx-good:latest
docker exec good_container ps aux

# Output:
# USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
# root         1  0.0  0.5  32456  5672 ?        Ss   10:00   0:00 nginx: master process
# 
# SUCCESS: nginx is PID 1

# Stop gracefully
docker stop good_container
# Stops in <1 second - nginx received SIGTERM and shut down gracefully
```

## Example 8: Docker Compose with Restart Policy
```yaml
# docker-compose.yml
version: '3.8'

services:
  nginx:
    build: ./nginx
    container_name: inception_nginx
    ports:
      - "443:443"
    # Container will restart automatically if it crashes
    # Only works properly if PID 1 exits correctly
    restart: always
    depends_on:
      - wordpress
    networks:
      - inception_net

  wordpress:
    build: ./wordpress
    container_name: inception_wordpress
    restart: always
    depends_on:
      - mariadb
    volumes:
      - wp_data:/var/www/html
    networks:
      - inception_net

  mariadb:
    build: ./mariadb
    container_name: inception_mariadb
    restart: always
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - inception_net

networks:
  inception_net:
    driver: bridge

volumes:
  wp_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /home/yourusername/data/wordpress
  
  db_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /home/yourusername/data/mariadb

# KEY POINT: restart: always only works if:
# 1. PID 1 exits properly (receives and handles signals)
# 2. The service actually terminates, not hangs forever
# 3. Docker can detect the container is down
```

# **Flashcards:**
---
CREATE 6 FLASHCARDS REGARDING THIS NOTE
This is the format: FRONT TEXT;; BACK TEXT

What is PID 1 in a Docker container and why is it special?;; PID 1 is the first process started in a container (whatever you specify as ENTRYPOINT/CMD). It's special because: (1) it must handle signals like SIGTERM for graceful shutdown, (2) it's responsible for reaping zombie processes, and (3) it has different signal handling than regular processes - the kernel won't terminate it on signals unless it explicitly handles them.

Why are commands like "tail -f /dev/null" and "sleep infinity" considered bad practice in Docker containers?;; These commands are anti-patterns because they become PID 1 but don't actually manage your service. Your real service (nginx, MariaDB) runs as a child process, meaning signals don't reach it properly, the container can't restart correctly, and it takes the full grace period (10 seconds) to stop. They violate Docker's single-process design philosophy.

What's the difference between exec form and shell form in Dockerfile CMD/ENTRYPOINT?;; Exec form: CMD ["nginx", "-g", "daemon off;"] runs the process directly as PID 1. Shell form: CMD nginx -g "daemon off;" wraps the command in /bin/sh -c, making the shell PID 1 and your service a child. Exec form is preferred because signals reach your application directly.

What does the "exec" command do in a bash script and why is it important for Docker entrypoints?;; The "exec" command replaces the current process (bash) with a new program instead of spawning it as a child. This is critical in Docker entrypoints because it makes your service (e.g., mysqld) become PID 1 after initialization, ensuring proper signal handling. Without exec, bash remains PID 1 and your service runs as a child.

Why must services like nginx and php-fpm run in "daemon off" or foreground mode in Docker?;; Daemon mode causes the service to fork into the background, which means the original process exits and returns control to the shell. In Docker, this would cause the container to exit immediately (if it's PID 1) or run as a background child (if a shell is PID 1). Foreground mode keeps the service as the main process, properly running as PID 1.

What happens when Docker sends SIGTERM to stop a container where PID 1 is not the actual service?;; When PID 1 is a shell or blocking command (like tail -f) instead of the service, SIGTERM goes to that process, not your actual service. The service doesn't receive the shutdown signal, can't clean up gracefully, and Docker waits the full grace period (default 10 seconds) before sending SIGKILL to forcefully terminate everything. This prevents graceful shutdown and proper cleanup.