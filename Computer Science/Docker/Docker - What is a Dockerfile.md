---
memory: to_finish
tags:
 - learned
language:
 - Docker
review-date: ""
last-reviewed: 2025-07-22
scheda: done
visit-count: 3
confidence-level: 1
consecutive-correct: 1
last-struggle-date: 2025-07-13
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
A Dockerfile solves the fundamental problem of application deployment consistency and environment reproducibility. It addresses the "it works on my machine" problem by creating a standardized, automated way to build identical application environments anywhere. Dockerfiles are crucial in modern software development because they enable Infrastructure as Code (IaC), allowing developers to version control their entire application stack, dependencies, and configuration. This ensures that applications run consistently across development, testing, and production environments, streamlines CI/CD pipelines, and eliminates deployment-related issues caused by environmental differences.

# **Core Explanation:**

---

A Dockerfile is a ==text file containing a sequence of instructions that Docker uses to automatically build a Docker image==. It serves as a blueprint or recipe that defines how to construct a containerized application environment from scratch.

Key characteristics:
- **Declarative format**: Uses simple, human-readable instructions to define the build process
- **Layer-based architecture**: <mark style="background:

# BBFABBA6;">Each instruction creates a new layer in the image,</mark> enabling efficient caching and reuse
- **Reproducible builds**: Same Dockerfile produces identical images across different environments
- **Version controllable**: Can be stored in source control alongside application code
- **Inheritance-based**: Can extend existing images using the FROM instruction
- **Automated execution**: Docker engine executes instructions sequentially during build process

How it works:
>1. **Base image selection**: Starts with a base image (FROM instruction)
>2. **Layer creation**: Each instruction creates a new read-only layer
>3. **Build context**: Docker sends the build context (directory contents) to the daemon
>4. **Instruction execution**: Docker executes each instruction in order
>5. **Image creation**: Final image is created with all layers combined
>6. **Metadata storage**: Image metadata (labels, exposed ports, etc.) is stored
>7. **Caching utilization**: Docker caches layers to speed up subsequent builds

Build process flow:
- Docker client sends build context to Docker daemon
- Daemon reads Dockerfile and executes each instruction
- Each instruction creates an intermediate container, runs the command, and commits the result as a new layer
- Final image is tagged and stored in local image registry

Common instructions:
- **FROM**: Specifies base image
- **RUN**: Executes commands during build
- **COPY/ADD**: Copies files from host to image
- **WORKDIR**: Sets working directory
- **ENV**: Sets environment variables
- **EXPOSE**: Documents exposed ports
- **CMD/ENTRYPOINT**: Defines default command to run

# **Related Concepts:**

---
**Docker Image**: The read-only template created by executing a Dockerfile. Images are the immutable artifacts that containers are instantiated from.

**Docker Container**: A running instance of a Docker image. Containers are created from images and add a writable layer on top.

**Base Image**: The starting point specified in the FROM instruction. Can be official images (ubuntu, node, python) or custom images.

**Docker Layer**: Each instruction in a Dockerfile creates a new layer. Layers are cached and reused to optimize build times and storage.

**Build Context**: The directory containing the Dockerfile and files to be included in the image. Sent to Docker daemon during build.

**Docker Registry**: Repository for storing and distributing Docker images (Docker Hub, ECR, etc.). Built images can be pushed to registries.

**Multi-stage Build**: Dockerfile technique using multiple FROM statements to optimize image size by separating build and runtime environments.

**Docker Compose**: Tool for defining multi-container applications using YAML files. Often references Dockerfiles for custom image builds.

**Container Orchestration**: Systems like Kubernetes that manage containerized applications at scale, using images built from Dockerfiles.

**.dockerignore**: File specifying which files/directories to exclude from build context, similar to .gitignore for Git.

**Image Layers**: Read-only filesystem layers that make up a Docker image. Each Dockerfile instruction creates a new layer.

**Container Runtime**: Software that executes containers (Docker Engine, containerd, etc.). Reads image metadata to run containers properly.

**Infrastructure as Code (IaC)**: Practice of managing infrastructure through code. Dockerfiles enable IaC for application environments.

**CI/CD Pipeline**: Automated build and deployment process. Dockerfiles integrate with CI/CD systems to build and deploy applications.

# **Examples:**

---
#

# Basic Dockerfile Examples:

#

#

# Simple Node.js Application Dockerfile:

```dockerfile

# Dockerfile for Node.js application

# FROM instruction specifies the base image to build upon

# node:16-alpine uses Node.js version 16 on Alpine Linux (smaller image)
FROM node:16-alpine

# LABEL adds metadata to the image for documentation and organization

# These labels help identify the image purpose and maintainer
LABEL maintainer="developer@example.com"
LABEL version="1.0"
LABEL description="Node.js web application"

# WORKDIR sets the working directory inside the container

# All subsequent commands will be executed from this directory

# Creates the directory if it doesn't exist
WORKDIR /usr/src/app

# COPY copies files from build context (host) to image filesystem

# Copy package.json and package-lock.json first for better caching

# This allows npm install to be cached if dependencies haven't changed
COPY package*.json ./

# RUN executes commands during the build process

# npm ci installs dependencies exactly as specified in package-lock.json

# More reliable than npm install for production builds
RUN npm ci --only=production

# COPY application source code to the image

# Done after npm install to leverage Docker layer caching

# If source code changes, only this layer and subsequent ones rebuild
COPY . .

# Create a non-root user for security

# Running containers as root is a security risk
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# Change ownership of application files to non-root user

# Ensures the application can read/write necessary files
RUN chown -R nextjs:nodejs /usr/src/app
USER nextjs

# EXPOSE documents which port the application listens on

# This is for documentation only - doesn't actually publish the port

# Use docker run -p to publish ports
EXPOSE 3000

# ENV sets environment variables that persist in the container

# These are available to the application at runtime
ENV NODE_ENV=production
ENV PORT=3000

# HEALTHCHECK defines how Docker should test if container is healthy

# Docker will run this command periodically to check container health
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
 CMD curl -f || exit 1

# CMD specifies the default command to run when container starts

# Can be overridden by providing command when running container

# Uses array format (exec form) which is preferred
CMD ["node", "server.js"]
````

#

#

# Multi-stage Build Example:

```dockerfile

# Multi-stage Dockerfile for React application

# Stage 1: Build stage - contains build tools and dependencies
FROM node:16-alpine AS builder

# Set working directory for build stage
WORKDIR /app

# Copy package files for dependency installation
COPY package*.json ./

# Install all dependencies (including dev dependencies for building)
RUN npm ci

# Copy source code for building
COPY . .

# Build the application for production

# Creates optimized, minified static files
RUN npm run build

# Stage 2: Production stage - minimal runtime environment
FROM nginx:alpine AS production

# Copy built application from builder stage to nginx directory

# --from=builder references the previous stage

# Only copies the built files, not the source code or node_modules
COPY --from=builder /app/build /usr/share/nginx/html

# Copy custom nginx configuration if needed
COPY nginx.conf /etc/nginx/nginx.conf

# Expose port 80 for HTTP traffic
EXPOSE 80

# Start nginx in foreground mode

# Containers must run processes in foreground to stay alive
CMD ["nginx", "-g", "daemon off;"]
```

#

#

# Python Flask Application Dockerfile:

```dockerfile

# Dockerfile for Python Flask application

# Use Python 3 slim image for smaller size
FROM python:3.9-slim

# Set environment variables to optimize Python behavior in containers

# PYTHONDONTWRITEBYTECODE prevents Python from writing .pyc files

# PYTHONUNBUFFERED ensures output is sent directly to terminal
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Install system dependencies required by Python packages

# Update package list and install required system packages
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
 gcc \
 && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy requirements file first for better caching

# If requirements.txt doesn't change, this layer will be cached
COPY requirements.txt .

# Install Python dependencies

# --no-cache-dir prevents caching of packages to reduce image size
RUN pip install --no-cache-dir --upgrade pip \
 && pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user for security
RUN useradd --create-home --shell /bin/bash app \
 && chown -R app:app /app
USER app

# Expose port for Flask application
EXPOSE 5000

# Set default environment for Flask
ENV FLASK_APP=app.py
ENV FLASK_ENV=production

# Health check for Flask application
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
 CMD curl -f || exit 1

# Default command to run the application
CMD ["python", "-m", "flask", "run", "--host=0.0.0"]
```

#

#

# Java Spring Boot Application Dockerfile:

```dockerfile

# Multi-stage Dockerfile for Java Spring Boot application

# Stage 1: Build stage using Maven
FROM maven:3.8-openjdk-17 AS build

# Set working directory
WORKDIR /app

# Copy Maven configuration files first

# This enables caching of dependency downloads
COPY pom.xml .
COPY src ./src

# Build the application

# -DskipTests skips running tests during build (run separately in CI)
RUN mvn clean package -DskipTests

# Stage 2: Runtime stage with minimal JRE
FROM openjdk:17-jre-slim

# Install curl for health checks
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# Create app directory and user
RUN mkdir /app && \
 groupadd -r spring && \
 useradd -r -g spring spring

# Set working directory
WORKDIR /app

# Copy built JAR from build stage
COPY --from=build /app/target/*.jar app.jar

# Change ownership to non-root user
RUN chown -R spring:spring /app
USER spring

# Expose application port
EXPOSE 8080

# JVM optimization for containers
ENV JAVA_OPTS="-Xmx512m -Xms512m"

# Health check endpoint
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
 CMD curl -f || exit 1

# Entry point for Spring Boot application
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

#

#

# Advanced Dockerfile with Build Arguments:

```dockerfile

# Advanced Dockerfile with build arguments and conditional logic

# ARG defines build-time variables that can be passed during build
ARG NODE_VERSION=16
ARG ENVIRONMENT=production

# Use build argument in FROM instruction
FROM node:${NODE_VERSION}-alpine

# Build argument for app version
ARG APP_VERSION=latest
ARG BUILD_DATE

# Set labels using build arguments
LABEL version=${APP_VERSION}
LABEL build-date=${BUILD_DATE}
LABEL environment=${ENVIRONMENT}

# Set working directory
WORKDIR /usr/src/app

# Copy package files
COPY package*.json ./

# Conditional installation based on environment

# Install dev dependencies only for development environment
RUN if [ "$ENVIRONMENT" = "development" ]; then \
 npm install; \
 else \
 npm ci --only=production; \
 fi

# Copy source code
COPY . .

# Build application if in production
RUN if [ "$ENVIRONMENT" = "production" ]; then \
 npm run build; \
 fi

# Create user
RUN addgroup -g 1001 -S nodejs && \
 adduser -S nextjs -u 1001

# Set ownership
RUN chown -R nextjs:nodejs /usr/src/app
USER nextjs

# Expose port
EXPOSE 3000

# Set environment variables
ENV NODE_ENV=${ENVIRONMENT}
ENV APP_VERSION=${APP_VERSION}

# Conditional CMD based on environment

# Development: start with nodemon for auto-restart

# Production: start with optimized node command
CMD if [ "$ENVIRONMENT" = "development" ]; then \
 npm run dev; \
 else \
 npm start; \
 fi
```

#

#

# Docker Build Commands and Usage:

```bash

# Basic build command

# -t tags the image with a name

# . specifies build context (current directory)
docker build -t my-app:latest .

# Build with build arguments

# --build-arg passes variables to ARG instructions
docker build \
 --build-arg NODE_VERSION=18 \
 --build-arg ENVIRONMENT=production \
 --build-arg APP_VERSION=1 \
 --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
 -t my-app:1 .

# Build targeting specific stage in multi-stage build

# --target specifies which stage to build
docker build --target builder -t my-app:builder .

# Build with no cache (force rebuild of all layers)

# --no-cache ignores cached layers
docker build --no-cache -t my-app:latest .

# Build with custom Dockerfile name

# -f specifies Dockerfile location
docker build -f Dockerfile.production -t my-app:prod .

# Build with build context from URL
docker build -t my-app .com/user/repo.git

# Build with progress output

# --progress shows detailed build progress
docker build --progress=plain -t my-app:latest .
```

#

#

# .dockerignore Example:

```dockerignore

# .dockerignore file excludes files from build context

# Reduces build time and image size

# Version control
.git
.gitignore

# Dependencies
node_modules
npm-debug.log*

# IDE files
.vscode
.idea
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db

# Build artifacts
dist/
build/
*.log

# Environment files
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# Documentation
README.md
docs/

# Test files
test/
spec/
*.test.js
*.spec.js

# Docker files (don't include other Dockerfiles)
Dockerfile*
docker-compose*.yml
```

#

#

# Best Practices Example:

```dockerfile

# Best practices Dockerfile example

# Use specific version tags for reproducible builds
FROM node:16.2-alpine3

# Set metadata
LABEL maintainer="team@example.com"
LABEL version="1.0"

# Install security updates
RUN apk update && apk upgrade && apk add --no-cache dumb-init

# Create app directory and user first
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# Set working directory
WORKDIR /usr/src/app

# Copy package files with proper ownership
COPY --chown=nextjs:nodejs package*.json ./

# Switch to non-root user early
USER nextjs

# Install dependencies
RUN npm ci --only=production && npm cache clean --force

# Copy source code
COPY --chown=nextjs:nodejs . .

# Remove unnecessary files
RUN rm -rf .git .gitignore README.md docs/

# Set environment variables
ENV NODE_ENV=production
ENV PORT=3000

# Expose port
EXPOSE 3000

# Add health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
 CMD node healthcheck.js

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]

# Start application
CMD ["node", "server.js"]
```

# **Flashcards:**

---
What is a Dockerfile and what problem does it solve?;; A Dockerfile is a text file containing instructions to build a Docker image. It solves the "it works on my machine" problem by creating reproducible, consistent application environments that run identically across different systems.

What are the key differences between RUN, CMD, and ENTRYPOINT instructions?;; RUN executes commands during build time and creates new layers. CMD sets the default command to run when container starts (can be overridden). ENTRYPOINT sets the command that always runs when container starts (cannot be overridden, only appended to).

What is the purpose of multi-stage builds in Docker?;; Multi-stage builds allow using multiple FROM statements to separate build and runtime environments. This reduces final image size by excluding build tools and dependencies from the production image while keeping the build process in the same Dockerfile.

Why should you copy package.json before copying application source code?;; Copying package.json first leverages Docker's layer caching. If source code changes but dependencies don't, the dependency installation layer remains cached, speeding up subsequent builds significantly.

What is the difference between COPY and ADD instructions?;; COPY simply copies files/directories from build context to image. ADD has additional features like automatic extraction of tar files and fetching files from URLs, but COPY is preferred for simple file copying as it's more predictable.

What security considerations should be followed when writing Dockerfiles?;; Use specific base image tags, run containers as non-root users, keep images minimal, scan for vulnerabilities, don't include secrets in layers, use .dockerignore to exclude sensitive files, and keep the attack surface small by installing only necessary packages.