---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-10-23
scheda: done
visit-count: 3
confidence-level: 2.5
consecutive-correct: 3
last-struggle-date: ""
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

# Purpose/Why:

---
The Same-Origin Policy (SOP) is a critical security mechanism ==implemented by web browsers to prevent malicious websites from accessing or manipulating data from another website==. It creates a protective boundary between different origins, preventing unauthorized cross-origin data access and protecting user privacy and security. ==Without SOP, malicious scripts could freely access sensitive information across domains, leading to data theft, session hijacking, and other security vulnerabilities.==

# Core Explanation:

---

The Same-Origin Policy restricts how a document or script loaded from one origin can interact with resources from another origin. ==An origin is defined by the combination of three elements:==
1. ==Protocol (e.g., HTTP vs. HTTPS)==
2. ==Domain (e.g., example.com)==
3. ==Port (e.g., 80, 443)==

Two URLs share the same origin only when all three components match exactly. Under this policy:

- Scripts can freely access data from their own origin
- Scripts are generally blocked from reading data from different origins
- Scripts can write data to different origins (like form submissions) but cannot read the response
- Embedding resources (images, CSS, scripts) from other origins is permitted
- Reading embedded resources from other origins is restricted

The policy applies to XMLHttpRequest, fetch API calls, DOM access across frames/iframes, and local storage access.

## Cross-Origin and Cross-Domain Requests

**Cross-Origin Request**: Occurs when a web page from one origin (domain, protocol, or port) requests resources from a different origin. For example, when a page at `.com` makes an AJAX request to `.different-domain.com`. Browsers restrict these requests due to the Same-Origin Policy.

**Cross-Domain Request**: Essentially the same as a cross-origin request, referring specifically to requests made from one domain to another. The term "cross-origin" is more technically precise since "origin" includes not just the domain, but also the protocol and port number.

# Related Concepts:

---
- **[[CORS (Cross-Origin Resource Sharing)]]**: A mechanism that allows servers to specify who can access their resources, relaxing the Same-Origin Policy in controlled ways. It uses HTTP headers to tell browsers whether to allow cross-origin requests.

- **Content Security Policy (CSP)**: A security standard that helps prevent attacks like XSS by controlling which resources a page can load. While SOP restricts access between origins, CSP restricts what resources can be loaded within an origin.

- **JSONP**: An older technique that bypasses SOP by using script tags to load data from different origins. It's largely superseded by CORS.

- **iframe Sandboxing**: A way to embed content from other origins with restricted permissions, enhancing the protections offered by SOP.

- **CSRF (Cross-Site Request Forgery)**: An attack that tricks users into making unwanted actions on websites where they're authenticated. SOP helps prevent CSRF by restricting cross-origin reads.

- **XSS (Cross-Site Scripting)**: An attack where malicious scripts are injected into trusted websites. While SOP doesn't directly prevent XSS, it limits what injected scripts can access.

# Examples:

---
```javascript
// CLIENT-SIDE EXAMPLES

// Example 1: Same-Origin Policy in action
// This fetch request to the same origin will succeed
// assuming the endpoint exists and returns valid data
fetch('/api/data')
 .then(response => response.json)
 .then(data => console.log('Same origin data:', data))
 .catch(error => console.error('Error:', error));

// Example 2: Cross-Origin request that will be blocked by SOP
// This request will fail unless the server at api.example.com
// has CORS headers configured to allow requests from our origin
fetch('.example.com/data')
 .then(response => response.json)
 .then(data => console.log('Cross-origin data:', data))
 .catch(error => console.error('This will likely fail due to SOP:', error));

// Example 3: Using iframes and SOP restrictions
// Create an iframe to another domain
const iframe = document.createElement('iframe');
iframe.src = '.com';
document.body.appendChild(iframe);

// After the iframe loads, try to access its content
iframe.onload = function {
 try {
 // This will fail due to SOP if the domains are different
 // We cannot access the content of a cross-origin iframe
 const iframeContent = iframe.contentDocument.body.innerHTML;
 console.log(iframeContent);
 } catch (error) {
 console.error('Cannot access cross-origin iframe content due to SOP:', error);
 }
};

// Example 4: Accessing localStorage across origins
// If we have two tabs open, one at example.com and one at example.org
// The following code running on example.com cannot access example.org's localStorage
function tryAccessOtherDomainStorage {
 try {
 // This would work if the iframe is from the same origin
 // But will fail for cross-origin iframes due to SOP
 const iframe = document.querySelector('iframe');
 const otherDomainStorage = iframe.contentWindow.localStorage;
 console.log(otherDomainStorage);
 } catch (error) {
 console.error('Cannot access localStorage from another origin:', error);
 }
}
```

# Flashcards:

---
What is the Same-Origin Policy?;; A security mechanism that restricts how a document or script loaded from one origin can interact with resources from another origin, preventing unauthorized cross-origin data access.

What defines an "origin" in the context of Same-Origin Policy?;; An origin is defined by the combination of three elements: protocol (HTTP/HTTPS), domain, and port number.

What's the difference between a cross-origin request and a cross-domain request?;; A cross-origin request refers to requests between different origins (protocol, domain, or port), while a cross-domain request specifically refers to different domains. Cross-origin is more technically precise.

Why is the Same-Origin Policy important for web security?;; It prevents malicious websites from accessing or manipulating data from another website, protecting against data theft, session hijacking, and other security vulnerabilities.

What types of cross-origin interactions does the Same-Origin Policy restrict?;; It restricts reading data from different origins but allows writing data (like form submissions), embedding resources (images, CSS, scripts), and restricts DOM access across frames/iframes.

How does CORS relate to the Same-Origin Policy?;; CORS (Cross-Origin Resource Sharing) is a mechanism that relaxes the Same-Origin Policy in controlled ways, allowing servers to specify who can access their resources using HTTP headers.