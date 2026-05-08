---
memory: to_finish
tags:
 - learned
language:
 - Docker
review-date: ""
last-reviewed: 2025-08-18
scheda: done
visit-count: 2
confidence-level: 1
consecutive-correct: 1
last-struggle-date: 2025-08-08
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
The `docker images` command solves the fundamental problem of **image inventory management** in containerized environments. Without a way to list and inspect local Docker images, developers would have no visibility into:
- Which images are available locally for container creation
- How much disk space images are consuming
- Which versions/tags of images exist on the system
- When images were created and their relationships

This visibility is crucial because:
- **Resource Management**: ==Docker images can consume significant disk space, and accumulate over time==
- **Debugging**: Developers need to verify which image versions are available when containers fail
- **Cleanup Operations**: Identify unused or outdated images for removal
- **Development Workflow**: Confirm successful image builds and track image history
- **System Maintenance**: Monitor image storage usage and maintain clean development environments

In production environments, this becomes even more critical for capacity planning and ensuring the correct image versions are deployed.

# **Core Explanation:**

---
**Docker Images Command** (`docker images` or `docker image ls`) <mark style="background:

# D2B3FFA6;">displays a list of all Docker images stored locally on the system</mark>. It provides essential metadata about each image without requiring network access.

**Key Characteristics:**
- **Local Inventory**: Shows only images stored locally, not registry contents
- **Metadata Display**: Provides repository, tag, image ID, creation date, and size
- **Multiple Formats**: Supports various output formats (table, JSON, custom templates)
- **Filtering Options**: Can filter results by repository, tag, or other criteria
- **No Network Required**: Works offline since it queries local Docker daemon

**How It Works:**
1. **Query Docker Daemon**: Command communicates with local Docker daemon
2. **Image Database**: Daemon maintains metadata database of all local images
3. **Layer Information**: Aggregates data from image layers and manifests
4. **Formatted Output**: Presents information in human-readable or machine-parseable formats

**Default Output Columns:**
- **REPOSITORY**: Image name (e.g., `nginx`, `ubuntu`)
- **TAG**: Version or label (e.g., `latest`, `1.0`, `alpine`)
- **IMAGE ID**: Unique identifier (first 12 characters of SHA256 hash)
- **CREATED**: When the image was built
- **SIZE**: Total size of image layers

**Common Use Cases:**
- Verify image availability before running containers
- Monitor disk usage and identify large images
- Check image creation dates and versions
- Prepare cleanup operations
- Debug container startup issues

# **Related Concepts:**

---
**Docker Image Layers**: Images are composed of read-only layers stacked on top of each other - `docker images` shows the total size of all layers combined.

**Image Tags**: Version labels that allow multiple names for the same image - `docker images` shows all tags pointing to each unique image.

**Image ID vs Digest**: Image ID is a local identifier shown by `docker images`, while digest is a content-addressable hash used in registries.

**Docker Image Inspect**: `docker inspect` provides detailed metadata about a specific image, while `docker images` gives overview information about all images.

**Docker Image History**: `docker history` shows the layer-by-layer build history of an image, complementing the summary view from `docker images`.

**Docker System DF**: Shows disk usage summary including images, containers, and volumes - provides context for the space usage shown in `docker images`.

**Docker Image Prune**: Removes unused images - often used after analyzing output from `docker images` to identify cleanup candidates.

**Registry vs Local Storage**: `docker images` shows local cache, while registry commands show remote available images.

# **Examples:**

---
```bash

# EXAMPLE 1: Basic Image Listing Commands

# List all local images (default format)
docker images

# Shows: REPOSITORY, TAG, IMAGE ID, CREATED, SIZE in table format

# Most commonly used command for quick image overview

# Alternative command with same result
docker image ls

# Newer syntax, functionally identical to 'docker images'

# Both commands show all locally stored images

# List images with full image IDs (not truncated)
docker images --no-trunc

# Shows complete SHA256 hashes instead of shortened versions

# Useful when you need the full image ID for scripting or debugging
````

```bash

# EXAMPLE 2: Filtering and Searching Images

# List images from specific repository
docker images nginx

# Shows only images with 'nginx' in the repository name

# Useful when you have multiple versions of the same application

# List images with specific tag
docker images ubuntu:20

# Shows only images matching exact repository:tag combination

# Returns single result if image exists, empty if not found

# List images using wildcard patterns
docker images "ubuntu:*"

# Shows all ubuntu images regardless of tag

# Quotes prevent shell expansion of the asterisk

# Filter images by creation date
docker images --filter "before=ubuntu:18.04"

# Shows images created before the specified image

# Useful for finding older versions that might need cleanup

# Filter dangling images (untagged)
docker images --filter "dangling=true"

# Shows images without repository:tag names

# These are often leftover from builds and safe to remove
```

```bash

# EXAMPLE 3: Custom Output Formatting

# Output in JSON format
docker images --format json

# Machine-readable output for scripts and automation

# Each image is a separate JSON object on its own line

# Custom table format with specific columns
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Shows only repository, tag, and size columns

# Useful for focusing on specific information

# Custom format for scripting
docker images --format "{{.Repository}}:{{.Tag}} {{.Size}}"

# Simple format perfect for parsing in shell scripts

# Each line contains image name and size

# Show only image IDs (useful for scripting)
docker images --format "{{.ID}}"

# Returns just the image IDs, one per line

# Commonly used in cleanup scripts: docker rmi $(docker images -q)
```

```bash

# EXAMPLE 4: Advanced Filtering and Analysis

# Combine multiple filters
docker images --filter "label=maintainer=nginx" --filter "before=2023-01-01"

# Shows nginx images with specific label created before date

# Demonstrates how multiple filters work together with AND logic

# List images sorted by size (requires additional processing)
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | sort -k3 -h

# Sorts images by size column using human-readable format

# Helps identify largest images consuming disk space

# Show only repository names (remove duplicates)
docker images --format "{{.Repository}}" | sort -u

# Lists unique repository names

# Useful for inventory of what applications you have images for

# Count images by repository
docker images --format "{{.Repository}}" | sort | uniq -c

# Shows how many images you have for each repository

# Helps identify repositories with many versions
```

```bash

# EXAMPLE 5: Practical Maintenance Scripts

# !/bin/bash

# Script to analyze and report on local Docker images

echo "=== Docker Images Summary ==="
echo "Total images: $(docker images --format "{{.ID}}" | wc -l)"
echo "Total size: $(docker images --format "{{.Size}}" | sed 's/[A-Za-z]//g' | awk '{sum += $1} END {print sum "MB (approximate)"}')"
echo ""

echo "=== Largest Images ==="

# Show top 5 largest images
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | head -6
echo ""

echo "=== Dangling Images (candidates for cleanup) ==="

# Show untagged images that can be safely removed
docker images --filter "dangling=true" --format "table {{.ID}}\t{{.CreatedAt}}\t{{.Size}}"
echo ""

echo "=== Images older than 30 days ==="

# Find old images that might be outdated
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.CreatedAt}}" | \
 awk 'NR>1 && $3 < "'$(date -d '30 days ago' +%Y-%m-%d)'"'

# This script provides comprehensive image analysis for maintenance decisions
```

```bash

# EXAMPLE 6: Integration with Container Operations

# Check if specific image exists before running container
if docker images nginx:alpine --format "{{.ID}}" | grep -q .; then
 echo "Image exists, starting container..."
 docker run -d nginx:alpine
else
 echo "Image not found, pulling first..."
 docker pull nginx:alpine
 docker run -d nginx:alpine
fi

# Prevents errors by ensuring image availability before container creation

# List images used by running containers
docker ps --format "{{.Image}}" | sort -u | while read image; do
 echo "=== $image ==="
 docker images "$image"
done

# Shows detailed information about images currently in use

# Helps identify which images are actively needed vs. cleanup candidates

# Find unused images (not referenced by any container)
docker images --format "{{.Repository}}:{{.Tag}}" | while read image; do
 if ! docker ps -a --format "{{.Image}}" | grep -q "^$image$"; then
 echo "Unused image: $image"
 fi
done

# Identifies images that have no associated containers

# Useful for finding images safe to remove during cleanup
```

# **Flashcards:**

---
What information does the `docker images` command display by default?;; Repository name, tag, image ID, creation date, and size in a table format showing all locally stored Docker images.

What is the difference between `docker images` and `docker image ls`?;; They are functionally identical - `docker image ls` is the newer syntax but both commands show the same list of local Docker images.

How do you show only dangling (untagged) images?;; Use `docker images --filter "dangling=true"` to show images without repository:tag names that are often leftover from builds.

What does the `--no-trunc` flag do with `docker images`?;; It displays the full SHA256 image IDs instead of the truncated 12-character versions shown by default.

How can you format `docker images` output for use in scripts?;; Use `--format "{{.ID}}"` to show only image IDs, or `--format json` for machine-readable JSON output, or custom formats like `{{.Repository}}:{{.Tag}}`.

What command combination helps identify the largest images consuming disk space?;; `docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"` and then sort by size column, or use `docker system df` for overall disk usage summary.