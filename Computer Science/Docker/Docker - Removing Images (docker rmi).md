---
memory: to_finish
tags:
  - learned
language:
  - Docker
review-date:
last-reviewed: 2025-09-09
scheda: done
visit-count: 4
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-08-18
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

The `docker rmi` command solves the fundamental problem of **image storage management** and **disk space optimization** in containerized environments. Without the ability to remove unused Docker images, systems would face:
- **Storage Exhaustion**: Docker images accumulate over time, consuming significant disk space
- **Development Clutter**: Multiple versions of images from builds and pulls create confusion
- **Performance Degradation**: Excessive images can slow down Docker operations and system performance
- **Resource Waste**: Unused images consume valuable storage resources unnecessarily

This is crucial because:
- **Cost Management**: In cloud environments, storage costs money and unused images waste resources
- **System Maintenance**: Clean systems are easier to manage and troubleshoot
- **Security**: Removing outdated images eliminates potential security vulnerabilities
- **Development Workflow**: Clean environments prevent confusion about which image versions are current
- **Automation**: CI/CD pipelines need to clean up intermediate images to prevent storage bloat

In production environments, proper image lifecycle management is essential for maintaining system health and controlling operational costs.

# **Core Explanation:**
---

**Docker RMI Command** (`docker rmi` or `docker image rm`) removes one or more Docker images from local storage. It's the primary tool for managing image lifecycle and freeing up disk space.

**Key Characteristics:**
- **Local Operation**: ==Only removes images from local storage, not from registries==
- **Reference-Based**: Can remove images by name, tag, or image ID
- **Dependency Checking**: Prevents removal of images used by existing containers
- **Layer Management**: Handles shared layers between images intelligently
- **Batch Operations**: Can remove multiple images in a single command
- **Force Options**: Can override safety checks when necessary

**How It Works:**
1. **Reference Resolution**: Resolves image names/tags to internal image IDs
2. **Dependency Check**: Verifies no containers are using the image
3. **Layer Analysis**: Determines which layers can be safely removed
4. **Storage Cleanup**: Removes image metadata and unreferenced layers
5. **Registry Sync**: Updates local image database

**Safety Mechanisms:**
- **Container Check**: Prevents removal of images used by existing containers
- **Multi-tag Protection**: Removing one tag doesn't delete image if other tags exist
- **Confirmation Prompts**: Shows what will be removed before deletion
- **Rollback**: No automatic rollback - removals are permanent

**Common Scenarios:**
- Cleanup after development builds
- Remove outdated image versions
- Free disk space when storage is limited
- Prepare systems for deployment
- Maintain clean development environments

# **Related Concepts:**
---

**Docker Image Layers**: Images consist of read-only layers - `docker rmi` removes layers only when no other images reference them, optimizing storage.

**Image Tags vs Image IDs**: Tags are human-readable names while IDs are unique identifiers - removing a tag doesn't delete the image if other tags exist.

**Dangling Images**: Untagged images created during builds - `docker rmi` with filters can remove these specifically.

**Docker Image Prune**: Automated cleanup command that removes unused images - more aggressive than selective `docker rmi` operations.

**Container Dependencies**: Running or stopped containers prevent image removal - `docker rmi` checks these dependencies before deletion.

**Docker System Prune**: Comprehensive cleanup including images, containers, networks, and volumes - `docker rmi` focuses only on images.

**Registry vs Local Storage**: `docker rmi` affects only local storage - images remain in registries and can be re-pulled.

**Image Manifest**: Metadata describing image contents - removed along with image data during `docker rmi`.

# **Examples:**
---

```bash
# EXAMPLE 1: Basic Image Removal Commands

# Remove a single image by name and tag
docker rmi nginx:alpine
# Removes the nginx:alpine image from local storage
# Fails if any containers (running or stopped) are using this image

# Remove image by Image ID
docker rmi 4f380adfc10f
# Uses the unique image ID instead of name:tag
# More precise when you have multiple tags pointing to same image

# Remove multiple images at once
docker rmi nginx:alpine ubuntu:20.04 python:3.9
# Space-separated list removes multiple images in one command
# Stops at first error if any image cannot be removed

# Force removal (bypass safety checks)
docker rmi -f nginx:alpine
# Forces removal even if containers are using the image
# Dangerous: can break running containers, use with caution
````

```bash
# EXAMPLE 2: Removing Images with Filters and Patterns

# Remove all dangling images (untagged)
docker rmi $(docker images -f "dangling=true" -q)
# First part gets IDs of dangling images
# Second part removes them all at once
# Safe operation since dangling images aren't used by containers

# Remove all images from a specific repository
docker rmi $(docker images "nginx" -q)
# Gets all nginx image IDs and removes them
# Will fail if any nginx containers exist

# Remove images older than a specific date
docker images --filter "before=ubuntu:18.04" -q | xargs docker rmi
# Finds images created before ubuntu:18.04
# Uses xargs to handle large lists of image IDs
# Useful for cleaning up old development images
```

```bash
# EXAMPLE 3: Safe Removal with Dependency Checking

#!/bin/bash
# Script for safe image removal with dependency checking

IMAGE_TO_REMOVE="myapp:old-version"

# Check if any containers are using this image
CONTAINERS_USING_IMAGE=$(docker ps -a --filter "ancestor=$IMAGE_TO_REMOVE" -q)

if [ -n "$CONTAINERS_USING_IMAGE" ]; then
    echo "Warning: Containers are using image $IMAGE_TO_REMOVE"
    echo "Container IDs: $CONTAINERS_USING_IMAGE"
    echo "Stop and remove these containers first:"
    echo "docker rm -f $CONTAINERS_USING_IMAGE"
    exit 1
else
    echo "No containers using $IMAGE_TO_REMOVE, safe to remove"
    docker rmi "$IMAGE_TO_REMOVE"
    echo "Image $IMAGE_TO_REMOVE removed successfully"
fi
# This script prevents accidental removal of images in use
```

```bash
# EXAMPLE 4: Bulk Cleanup Operations

# Remove all unused images (not just dangling)
docker image prune -a
# More aggressive than docker rmi - removes all unused images
# Interactive prompt asks for confirmation before deletion

# Remove unused images without confirmation
docker image prune -a -f
# Same as above but skips confirmation prompt
# Useful in automated scripts and CI/CD pipelines

# Custom cleanup: remove images older than 24 hours
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.CreatedAt}}\t{{.ID}}" | \
    awk 'NR>1 && $3 < "'$(date -d '1 day ago' +%Y-%m-%d)'"' | \
    awk '{print $4}' | \
    xargs -r docker rmi
# Complex pipeline that:
# 1. Lists images with creation dates
# 2. Filters images older than 24 hours
# 3. Extracts image IDs
# 4. Removes them (if any exist)
```

```bash
# EXAMPLE 5: Development Workflow Integration

#!/bin/bash
# Development environment cleanup script

echo "=== Docker Image Cleanup ==="

# Show current disk usage
echo "Current Docker disk usage:"
docker system df

echo -e "\n=== Removing dangling images ==="
# Remove build artifacts and intermediate images
DANGLING_COUNT=$(docker images -f "dangling=true" -q | wc -l)
if [ "$DANGLING_COUNT" -gt 0 ]; then
    echo "Removing $DANGLING_COUNT dangling images..."
    docker rmi $(docker images -f "dangling=true" -q)
else
    echo "No dangling images found"
fi

echo -e "\n=== Removing unused development images ==="
# Remove images tagged with 'dev' or 'test' that aren't used by containers
docker images --format "{{.Repository}}:{{.Tag}}" | grep -E "(dev|test)" | while read image; do
    # Check if image is used by any container
    if ! docker ps -a --format "{{.Image}}" | grep -q "^$image$"; then
        echo "Removing unused development image: $image"
        docker rmi "$image" 2>/dev/null || echo "  Failed to remove $image"
    fi
done

echo -e "\n=== Final disk usage ==="
docker system df
# Complete cleanup workflow for development environments
```

```bash
# EXAMPLE 6: Advanced Removal Scenarios

# Remove all versions of an image except the latest
REPOSITORY="myapp"
# Get all tags except 'latest'
docker images "$REPOSITORY" --format "{{.Repository}}:{{.Tag}}" | \
    grep -v ":latest$" | \
    xargs -r docker rmi
# Keeps only the latest version, removes all other tags
# Useful for cleanup after multiple version builds

# Remove images by size (remove largest first)
docker images --format "table {{.Size}}\t{{.Repository}}:{{.Tag}}" | \
    sort -hr | \
    head -5 | \
    awk '{print $2}' | \
    while read image; do
        echo "Removing large image: $image"
        docker rmi "$image" 2>/dev/null || echo "  Cannot remove $image (in use)"
    done
# Targets largest images first to free maximum disk space
# Skips images that are in use by containers

# Conditional removal based on image labels
docker images --filter "label=environment=staging" -q | \
    xargs -r docker rmi
# Removes images with specific labels
# Useful for environment-specific cleanup in CI/CD

# Remove all images except those matching a pattern
docker images --format "{{.Repository}}:{{.Tag}}" | \
    grep -v -E "(nginx|postgres|redis)" | \
    xargs -r docker rmi
# Removes all images except core infrastructure images
# Preserves essential images while cleaning up application images
```

# **Flashcards:**
---

What is the primary difference between `docker rmi` and `docker image prune`?;; `docker rmi` removes specific images by name/ID, while `docker image prune` automatically removes all unused images without requiring specific image references.

What happens when you try to remove an image that's being used by a container?;; Docker prevents the removal and shows an error message. You must stop and remove the container first, or use the `-f` (force) flag to override this safety check.

What's the difference between removing an image by tag vs by Image ID?;; Removing by tag only removes that specific tag reference, while removing by Image ID removes the entire image. If multiple tags point to the same image, removing one tag won't delete the image data.

How can you safely check if an image is being used before removing it?;; Use `docker ps -a --filter "ancestor=image:tag" -q` to check if any containers (running or stopped) are using the image before attempting removal.

What does the `-f` flag do with `docker rmi` and when should you use it?;; The `-f` flag forces removal even if containers are using the image, potentially breaking those containers. Use it only when you're certain the running containers should be broken or in cleanup scripts.