---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-09-18
scheda: done
visit-count: 6
confidence-level: 2.5
consecutive-correct: 3
last-struggle-date: 2025-08-17
cssclasses:
  - difficult
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
Nginx solves the fundamental problem of efficiently ==serving web content and handling high-concurrency network connections.== High concurrency refers to **the ability of a system or application to receive a large number of concurrent requests in the same period**. It addresses the C10K problem (handling 10,000+ concurrent connections) through its ==event-driven, asynchronous architecture. Nginx is crucial in modern web infrastructure because it can serve static content extremely efficiently, act as a reverse proxy to distribute requests across multiple backend servers, perform load balancing, handle SSL termination, and provide caching capabilities.== Its low memory footprint and high performance make it essential for scalable web applications and microservices architectures.

In the<mark style="background: #BBFABBA6;"> context of Docker</mark> and containerization, <mark style="background: #BBFABBA6;">Nginx becomes even more critical as it serves as the gateway and orchestrator for containerized applications, enabling seamless scaling, service discovery, and traffic management in distributed systems.</mark>

# **Core Explanation:**
---
Nginx (pronounced "engine-x") <mark style="background: #FF5582A6;">is a high-performance web server, reverse proxy, and load balancer.</mark> It uses an event-driven, asynchronous, non-blocking architecture that allows it to handle thousands of concurrent connections with minimal resource consumption.

Key characteristics:
>- **Event-driven architecture**: Uses a master process with multiple worker processes, each handling thousands of connections in a single thread
>- **[[Non-blocking I⧸O]]**: Doesn't wait for slow operations, allowing efficient resource utilization
>- **Modular design**: Functionality is provided through modules that can be compiled in or loaded dynamically
>- **Configuration-based**: Uses declarative configuration files with a hierarchical structure
>- **High performance**: Optimized for serving static content and proxying requests

How it works:
1. ==Master process reads configuration and manages worker processes==
2. <mark style="background: #BBFABBA6;">Worker processes handle client connections using an event loop</mark>
3. <mark style="background: #ADCCFFA6;">Each connection is processed asynchronously without blocking other connections</mark>
4. <mark style="background: #D2B3FFA6;">Requests are handled based on configuration</mark> directives in server blocks
5. <mark style="background: #FFB8EBA6;">Can serve static files directly or proxy requests to backend applications</mark>

Common use cases:
- Static file serving
- Reverse proxy for web applications
- Load balancing across multiple servers
- SSL/TLS termination
- Caching and compression
- Rate limiting and security filtering

In Docker environments, Nginx excels as:
- **Container orchestration gateway**: Routes traffic to appropriate containerized services
- **Service mesh entry point**: Handles external traffic before distributing to internal services
- **Dynamic load balancer**: Automatically discovers and balances load across container instances
- **SSL termination point**: Centralizes certificate management for all containerized services

# **Related Concepts:**
---

**Apache HTTP Server**: Traditional web server using process/thread-per-connection model. Nginx is more resource-efficient for high-concurrency scenarios but Apache offers more modules and flexibility.

**Reverse Proxy**: Nginx acts as an intermediary between clients and backend servers, forwarding requests and returning responses. Different from forward proxy which serves clients.

**Load Balancer**: Nginx distributes incoming requests across multiple backend servers using various algorithms (round-robin, least connections, IP hash).

**Event Loop**: The core mechanism Nginx uses to handle multiple connections efficiently in a single thread, similar to Node.js event loop.

**Upstream**: Configuration block defining backend servers that Nginx can proxy requests to.

**Server Blocks**: Nginx equivalent of Apache virtual hosts, allowing multiple websites/applications on a single server.

**SSL Termination**: Process of decrypting SSL/TLS traffic at the proxy level before forwarding to backend servers.

**Caching**: Nginx can cache responses from backend servers to improve performance and reduce load.

**Rate Limiting**: Controlling the number of requests from clients to prevent abuse and ensure fair resource usage.

**Docker Containers**: Lightweight, portable application packages that include code, runtime, libraries, and dependencies. Nginx often runs in containers and routes traffic to other containerized services.

**Container Orchestration**: Systems like Docker Compose, Kubernetes, or Docker Swarm that manage multiple containers. Nginx serves as the ingress controller or load balancer.

**Service Discovery**: Mechanism for containers to find and communicate with each other. Nginx can be configured to automatically discover new service instances.

**Microservices Architecture**: Design pattern where applications are built as a collection of small, independent services. Nginx acts as the API gateway routing requests to appropriate microservices.

**Docker Networks**: Virtual networks that allow containers to communicate. Nginx containers connect to these networks to reach backend services.

**Health Checks**: Monitoring mechanism to ensure services are operational. Nginx can perform health checks on containerized services and route traffic only to healthy instances.

**Blue-Green Deployment**: Deployment strategy using two identical production environments. Nginx can switch traffic between them for zero-downtime deployments.

**Container Registry**: Repository for storing container images. Nginx official images are available on Docker Hub for easy deployment.

# **Examples:**
---
## Basic Nginx Configuration Examples:

```nginx
# Main configuration file structure (/etc/nginx/nginx.conf)
# Global directives affecting the entire server

# User and group that worker processes run as
# Important for security and file permissions
user nginx;

# Number of worker processes (usually set to number of CPU cores)
# Auto detects optimal number based on available cores
worker_processes auto;

# Maximum number of connections per worker process
# Total connections = worker_processes * worker_connections
events {
    worker_connections 1024;
    # Use efficient connection processing method on Linux
    use epoll;
}

# HTTP configuration block contains all web server settings
http {
    # Include MIME types for proper content type headers
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Optimize file serving by using sendfile system call
    # Reduces CPU usage and improves performance for static files
    sendfile on;
    
    # Keep connections alive to reduce overhead
    # Allows multiple requests over single connection
    keepalive_timeout 65;
    
    # Enable gzip compression to reduce bandwidth usage
    # Compresses text-based content before sending to client
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    
    # Include additional configuration files
    include /etc/nginx/conf.d/*.conf;
}
````

## Docker-specific Nginx Examples:

### Docker Compose with Nginx as Reverse Proxy:

```yaml
# docker-compose.yml
# Complete multi-service application with Nginx as reverse proxy
version: '3.8'

services:
  # Nginx service acting as reverse proxy and load balancer
  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    ports:
      # Map container port 80 to host port 80
      # This makes the application accessible from outside
      - "80:80"
      - "443:443"
    volumes:
      # Mount custom nginx configuration
      # This allows us to customize nginx behavior
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      # Nginx depends on backend services being available
      # Docker Compose starts dependencies first
      - web-app-1
      - web-app-2
      - api-service
    networks:
      # Connect nginx to custom network for service communication
      - app-network
    restart: unless-stopped

  # First instance of web application
  web-app-1:
    build: ./web-app
    container_name: web-app-1
    # No ports exposed to host - only accessible through nginx
    expose:
      - "3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
    networks:
      - app-network
    restart: unless-stopped

  # Second instance of web application for load balancing
  web-app-2:
    build: ./web-app
    container_name: web-app-2
    expose:
      - "3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
    networks:
      - app-network
    restart: unless-stopped

  # API service running on different port
  api-service:
    build: ./api
    container_name: api-service
    expose:
      - "8080"
    environment:
      - NODE_ENV=production
      - PORT=8080
    networks:
      - app-network
    restart: unless-stopped

  # Database service (not directly accessed by nginx)
  database:
    image: postgres:13
    container_name: postgres-db
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    volumes:
      # Persist database data
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

# Custom network for service communication
networks:
  app-network:
    driver: bridge

# Named volume for database persistence
volumes:
  postgres_data:
```

### Nginx Configuration for Docker Environment:

```nginx
# nginx.conf for Docker containerized applications
events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Upstream blocks define containerized services
    # Container names are used as hostnames in Docker networks
    upstream web_app {
        # Docker container names resolve to IP addresses automatically
        # Docker's internal DNS handles service discovery
        server web-app-1:3000 weight=1;
        server web-app-2:3000 weight=1;
        
        # Health checks for container instances
        # Mark as down after 2 failures, retry after 30 seconds
        server web-app-1:3000 max_fails=2 fail_timeout=30s;
        server web-app-2:3000 max_fails=2 fail_timeout=30s;
    }
    
    upstream api {
        # Single API service - could be scaled later
        server api-service:8080;
    }
    
    server {
        listen 80;
        server_name localhost;
        
        # Main web application routes
        location / {
            # Proxy to load-balanced web application containers
            proxy_pass http://web_app;
            
            # Important headers for containerized applications
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Connection settings optimized for containers
            proxy_connect_timeout 5s;
            proxy_send_timeout 10s;
            proxy_read_timeout 10s;
            
            # Enable connection keep-alive for better performance
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }
        
        # API routes go to API service container
        location /api/ {
            # Remove /api prefix when forwarding to backend
            rewrite ^/api/(.*)$ /$1 break;
            proxy_pass http://api;
            
            # API-specific headers
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            
            # CORS headers for API access
            add_header Access-Control-Allow-Origin *;
            add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
            add_header Access-Control-Allow-Headers "Origin, X-Requested-With, Content-Type, Accept, Authorization";
        }
        
        # Health check endpoint for container orchestration
        location /health {
            access_log off;
            return 200 "nginx healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
```

### Custom Nginx Dockerfile:

```dockerfile
# Dockerfile for custom Nginx container
# Use official nginx image as base
FROM nginx:alpine

# Copy custom configuration file
# This replaces the default nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf

# Copy static files if serving static content
COPY ./static /usr/share/nginx/html/static

# Copy SSL certificates for HTTPS
COPY ./ssl /etc/nginx/ssl

# Create custom index page
RUN echo '<h1>Custom Nginx Container</h1>' > /usr/share/nginx/html/index.html

# Expose port 80 for HTTP traffic
# This is documented but doesn't actually publish the port
EXPOSE 80

# Optional: Add custom startup script
COPY docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh

# Use custom entrypoint that can modify config at runtime
ENTRYPOINT ["/docker-entrypoint.sh"]

# Default command runs nginx in foreground
# Containers must run processes in foreground to stay alive
CMD ["nginx", "-g", "daemon off;"]
```

### Advanced Docker Compose with Auto-scaling:

```yaml
# docker-compose.yml with scaling capabilities
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      # Template configuration for dynamic upstream generation
      - ./nginx.conf.template:/etc/nginx/nginx.conf.template:ro
    depends_on:
      - web-app
    networks:
      - app-network
    deploy:
      # Keep nginx running even if it crashes
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 3

  # Web application service that can be scaled
  web-app:
    build: ./web-app
    expose:
      - "3000"
    networks:
      - app-network
    deploy:
      # Configure scaling parameters
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      restart_policy:
        condition: on-failure

  # Service discovery helper for dynamic configuration
  nginx-gen:
    image: nginxproxy/nginx-proxy:latest
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - ./nginx.tmpl:/etc/nginx/nginx.tmpl:ro
    networks:
      - app-network
    environment:
      - DEFAULT_HOST=localhost

networks:
  app-network:
    driver: overlay
    attachable: true
```

### Kubernetes Nginx Ingress Example:

```yaml
# nginx-ingress.yaml
# Nginx as Kubernetes Ingress Controller
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    # Use nginx as ingress controller
    kubernetes.io/ingress.class: "nginx"
    # Enable SSL redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # Custom nginx configuration
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Accept-Encoding "";
      sub_filter_once off;
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      # Route to web application service
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80
      # Route API requests to API service
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080

---
# ConfigMap for custom nginx configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    worker_processes auto;
    events {
        worker_connections 1024;
    }
    http {
        upstream web_app {
            server web-app-service:80;
        }
        upstream api {
            server api-service:8080;
        }
        server {
            listen 80;
            location / {
                proxy_pass http://web_app;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
            }
            location /api/ {
                proxy_pass http://api;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
            }
        }
    }
```

### Docker Swarm with Nginx Load Balancer:

```yaml
# docker-stack.yml for Docker Swarm
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - frontend
      - backend
    deploy:
      # Deploy on manager nodes for stability
      placement:
        constraints:
          - node.role == manager
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure

  web-app:
    image: myapp:latest
    networks:
      - backend
    deploy:
      replicas: 5
      update_config:
        parallelism: 2
        delay: 5s
        order: start-first
      restart_policy:
        condition: on-failure
    environment:
      - NODE_ENV=production

networks:
  frontend:
    external: true
  backend:
    driver: overlay
    attachable: true
```

### Dynamic Configuration with Consul Template:

```dockerfile
# Dockerfile for dynamic nginx configuration
FROM nginx:alpine

# Install consul-template for dynamic configuration
RUN apk add --no-cache consul-template

# Copy configuration template
COPY nginx.conf.tpl /etc/nginx/nginx.conf.tpl

# Copy startup script
COPY start.sh /start.sh
RUN chmod +x /start.sh

# Expose port
EXPOSE 80

# Use custom startup script
CMD ["/start.sh"]
```

```bash
#!/bin/sh
# start.sh - Dynamic configuration startup script

# Start consul-template in background
# This watches for service changes and updates nginx config
consul-template \
    -template="/etc/nginx/nginx.conf.tpl:/etc/nginx/nginx.conf:nginx -s reload" \
    -consul-addr="consul:8500" &

# Start nginx in foreground
nginx -g "daemon off;"
```

```nginx
# nginx.conf.tpl - Template for dynamic upstream configuration
events {
    worker_connections 1024;
}

http {
    {{range services}}
    upstream {{.Name}} {
        {{range service .Name}}
        server {{.Address}}:{{.Port}};
        {{end}}
    }
    {{end}}
    
    server {
        listen 80;
        
        {{range services}}
        location /{{.Name}}/ {
            proxy_pass http://{{.Name}};
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        {{end}}
    }
}
```

# **Flashcards:**
---

What is the main architectural advantage of Nginx over traditional web servers like Apache?;; Nginx uses an event-driven, asynchronous, non-blocking architecture that can handle thousands of concurrent connections with minimal resource consumption, while Apache uses a process/thread-per-connection model.

What are the two main processes types in Nginx and their roles?;; Master process (reads configuration and manages worker processes) and Worker processes (handle client connections using event loops and process requests asynchronously).

How does Docker's internal DNS help Nginx communicate with other containers?;; Docker's internal DNS automatically resolves container names to IP addresses within the same network, allowing Nginx to use container names as hostnames in upstream blocks (e.g., server web-app-1:3000).

What is the purpose of the 'expose' directive in Docker Compose versus 'ports'?;; 'expose' makes ports available only to other containers on the same network (internal communication), while 'ports' maps container ports to host ports making them accessible from outside the container.

In a Docker environment, why is it important to run Nginx with 'daemon off;'?;; Containers must run processes in the foreground to stay alive. If Nginx runs as a daemon (background process), the container would exit immediately because there's no foreground process keeping it running.

What are the key benefits of using Nginx as a reverse proxy in containerized microservices?;; Service discovery and load balancing, SSL termination, centralized routing, health checks, rate limiting, and the ability to scale backend services independently while maintaining a single entry point.