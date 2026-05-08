---
memory: to_finish
tags:
 - learned
language:
 - Core Concepts
review-date: ""
last-reviewed: 2025-08-08
scheda: done
visit-count: 3
confidence-level: 1
consecutive-correct: 1
last-struggle-date: 2025-07-20
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

# Purpose/Why:

---
URIs and URLs solve the fundamental problem of resource identification and location on the internet. They provide standardized ways to reference and access resources in distributed systems. These concepts are crucial in computer science because they enable the web's hypertext structure, allowing systems to link to and retrieve resources across networks in a consistent manner.

# Core Explanation:

---
#

#

# URI (Uniform Resource Identifier)
A URI is a ==string of characters that identifies a resource, either by name, location, or both==. It serves as the broader concept that encompasses both URLs and URNs (Uniform Resource Names).

Key characteristics of URIs:
- Identifies resources through a standardized format
- Can specify a resource without necessarily providing access methods
- Acts as a superclass that includes both URLs and URNs
- Follows a specific syntax: `scheme:[//authority]path[?query][

# fragment]`

#

#

# URL (Uniform Resource Locator)
A URL is a specific type of URI that ==not only identifies a resource but also specifies the mechanism for accessing it (the location)==.

Key characteristics of URLs:
- Subset of URI that includes access methods
- <mark style="background:

# BBFABBA6;">Specifies both what the resource is and how to get it</mark>
- Always includes a protocol (like http:, https:, ftp:)
- Example: .com/page.html

#

#

# Relationship
- All URLs are URIs, but not all URIs are URLs
- <mark style="background:

# FF5582A6;">URIs focus on identification, URLs focus on location and access method</mark>
- A URI without an access protocol is typically a URN, not a URL

# Related Concepts:

---
#

#

# URN (Uniform Resource Name)
A type of URI that identifies a resource by name in a particular namespace, without specifying location or how to access it. Example: `urn:isbn:0451450523`

#

#

# HTTP Methods
Operations that work on URIs to interact with web resources, defining how clients request actions on identified resources.

#

#

# Web APIs and Endpoints
Specific URLs that accept certain HTTP methods to perform defined operations, forming the interface for web services and RESTful APIs.

#

#

# Request/Response Cycle
The communication pattern where a client sends an HTTP request with a specific method to a URI and receives an HTTP response from the server.

#

#

# [[DNS (Domain Name System)]]
Translates human-readable domain names (part of URLs) into IP addresses that computers use to identify each other.

# Examples:

---
```javascript
// EXAMPLE 1: Basic URI/URL Components
const myURL = new URL('.example.com:8080/path/resource.html?query=value&param=data

# section');

// Scheme (protocol): https
console.log(myURL.protocol); // "https:"

// Authority (host + port): www.example.com:8080
console.log(myURL.host); // "www.example.com:8080"

// Path: /path/resource.html
console.log(myURL.pathname); // "/path/resource.html"

// Query: ?query=value&param=data
console.log(myURL.search); // "?query=value&param=data"

// Fragment:

# section
console.log(myURL.hash); // "

# section"

// EXAMPLE 2: Working with URLs in a fetch request
async function fetchResource(url) {
 // The URL string identifies the resource and how to access it
 const response = await fetch(url);

 // We can construct new URLs based on relative paths
 const baseURL = new URL('.example.com/v1/');
 const resourceURL = new URL('products/123', baseURL);

 console.log(resourceURL.href); // ".example.com/v1/products/123"
}

// EXAMPLE 3: Difference between URI and URL in practical usage
// This is a URL (it specifies the protocol/access method)
const documentURL = '.example.com/guide.pdf';

// This is a URI but not a URL (it identifies but doesn't specify how to access)
const documentURI = 'urn:doc:guide-v2';

// This function demonstrates that URIs can be used for identification
// while URLs are needed for actual retrieval
function useURIandURL {
 // We use the URI as a stable identifier that doesn't change
 const stableId = 'urn:doc:guide-v2';

 // We map the stable URI to an actual URL when we need to access it
 const uriToURLMap = {
 'urn:doc:guide-v2': '.example.com/v2/guide.pdf'
 };

 // Get the URL from our mapping when we need to access the resource
 const accessURL = uriToURLMap[stableId];

 return accessURL;
 // Returns: '.example.com/v2/guide.pdf'
}
```

# Flashcards:

---
What is a URI?;; A Uniform Resource Identifier is a string of characters that identifies a resource either by name, location, or both, serving as a standardized way to reference resources on the internet.

What is the relationship between URI and URL?;; A URL is a type of URI. All URLs are URIs, but not all URIs are URLs. URLs specifically provide the location and access method for a resource, while URIs may only identify a resource without specifying how to access it.

What components make up a typical URL?;; A typical URL consists of: scheme (protocol), authority (host + optional port), path, optional query parameters, and an optional fragment identifier.

How does a URN differ from a URL?;; A URN (Uniform Resource Name) identifies a resource by name within a specific namespace without specifying its location or access method, while a URL provides both identification and access method.

What's an example of a URI that is not a URL?;; urn:isbn:0451450523 is a URI (specifically a URN) that identifies a book by its ISBN number but doesn't specify how to access it.

Why are URIs important in web architecture?;; URIs provide a standardized way to identify resources across the internet, enabling linking between resources, consistent reference methods, and forming the foundation of the web's hypertext structure.