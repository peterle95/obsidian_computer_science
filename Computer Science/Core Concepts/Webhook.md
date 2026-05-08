---
memory: to_finish
tags:
  - to_learn
language:
  - Core Concepts
review-date: 2025-11-20
last-reviewed: 2025-10-13
scheda: done
visit-count: 1
confidence-level: 1
consecutive-correct: 0
last-struggle-date: 2025-10-13
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

The fundamental problem webhooks ==solve is **inefficient communication** between web services==. ==Traditionally, if an application (client) needs to know about an event in another application (server), it must constantly ask, or **"poll,"** the server: "Has anything happened yet? ... Has anything happened yet?". This is resource-intensive and creates delays.==

==Webhooks reverse this pattern. Instead of the client *pulling* information, the server **pushes** information to the client the instant an event occurs==.

**Why it's important:**
* **Efficiency**: It e<mark style="background: #BBFABBA6;">liminates the need for constant, often empty, polling requests</mark>, saving bandwidth and server resources for both parties.
* **Real-Time Updates**: It enables <mark style="background: #BBFABBA6;">applications to react to events instantly, which is crucial for things like chat notifications, payment confirmations, and continuous integration pipelines</mark>.
* **Automation**: They are the <mark style="background: #BBFABBA6;">backbone of modern automation, connecting disparate services to create seamless workflows</mark> (e.g., when a new sale happens in Stripe, automatically post a message in Slack).

# **Core Explanation:**
---

A **webhook** is a <mark style="background: #ABF7F7A6;">mechanism for one application to provide other applications with real-time information.</mark> It's essentially a ==**user-defined HTTP callback** (or a "reverse API")==. When a specific <mark style="background: #ABF7F7A6;">event happens in a source application, it automatically sends an HTTP POST request to a URL you've provided—the "webhook endpoint." This request contains a "payload," usually in JSON format, with details about the event.</mark>

Think of it <mark style="background: #FF5582A6;">like a magazine subscription: instead of you going to the newsstand every day to check for a new issue (polling), you give the publisher your address (the webhook URL), and they mail you the magazine as soon as it's published (the event occurs).</mark>

**How it works in 3 steps:**
1.  **Register:** You, the user or client application, register a URL (your endpoint) with the service that will be sending the updates (e.g., GitHub, Stripe, Shopify). You also specify which event(s) you're interested in (e.g., `git push`, `new payment`).
2.  **Event Occurs:** The specified event happens within the service. For example, a developer pushes code to a GitHub repository.
3.  **HTTP POST Request:** The service automatically compiles a data payload (e.g., information about the commit, the author, the branch) and sends it as an HTTP POST request to your registered URL. Your application, listening at that endpoint, can then process this data.

# **Related Concepts:**
---
* **API (Application Programming Interface):** Webhooks are a type of API, but they differ in their communication model. A standard API is **pull-based** (the client initiates a request for data), whereas a webhook is **push-based** (the server initiates the request when data is available).
* **Polling:** The direct alternative to webhooks. It involves a client repeatedly making API requests to a server at a set interval to check for updates. Webhooks are far more efficient.
* **Callback:** A familiar concept in programming where a function is passed as an argument to another, to be executed later. A webhook is the same idea applied to the web: it's an **HTTP callback**.
* **Pub/Sub (Publish/Subscribe):** A messaging pattern where publishers send messages to channels without knowing the subscribers. Subscribers express interest in channels and receive messages sent to them. Webhooks are a simple, common implementation of the Pub/Sub pattern over HTTP.

## **Webhooks in GitHub Actions and CI/CD**
---

Webhooks are fundamental to modern **Continuous Integration/Continuous Deployment (CI/CD)** pipelines.

* **Trigger:** A developer pushes a commit to a GitHub repository.
* **Event:** This `git push` is the event that GitHub is watching for.
* **Webhook Action:** ==GitHub (the provider) sends a webhook with a detailed JSON payload (containing commit info, changed files, author, etc.) to a pre-configured URL.==
* **Endpoint:** This ==URL belongs to a CI/CD server like GitHub Actions, Jenkins, or CircleCI.==
* **Result:** The CI/CD server receives the payload, understands that new code has been pushed, and automatically triggers a pre-defined workflow:
    1.  Build the application.
    2.  Run automated tests.
    3.  If tests pass, deploy the application to a staging or production server.

This entire process is automated and event-driven, all thanks to a simple webhook. It allows teams to test and deploy code changes instantly and reliably.

# **Examples:**
---

This example shows how to create a simple webhook *listener* or *endpoint* using Python and the [[Flask (web framework)]]. This server will wait for an external service (like GitHub) to send it a POST request.

```python
# main.py

# Import the necessary components from the Flask library.
# - Flask is the main class for creating our web application.
# - request allows us to access incoming request data (like headers and JSON payloads).
# - jsonify is a helper to convert Python dictionaries to JSON responses.
from flask import Flask, request, jsonify
import json

# Create an instance of the Flask web application.
# '__name__' is a special Python variable that gives Flask a name for the application.
app = Flask(__name__)

# Define a route for our webhook endpoint.
# This tells Flask that any HTTP request to '/webhook' should be handled by this function.
# methods=['POST'] specifies that this endpoint only accepts POST requests, which is standard for webhooks.
@app.route('/webhook', methods=['POST'])
def github_webhook():
    # Check if the incoming request has a JSON body.
    # Webhook payloads are almost always sent as JSON.
    if request.is_json:
        # Parse the JSON payload from the request into a Python dictionary.
        payload = request.get_json()

        # For demonstration, we'll print the payload to the console.
        # In a real application, you would process this data.
        # For example, if it's a GitHub push event, you might trigger a script.
        print("Received webhook payload:")
        # Use json.dumps for pretty-printing the JSON with indentation.
        print(json.dumps(payload, indent=4))

        # --- Your Custom Logic Goes Here ---
        # Example: Check the event type from GitHub's header
        github_event = request.headers.get('X-GitHub-Event')
        if github_event == 'push':
            pusher_name = payload.get('pusher', {}).get('name', 'Unknown')
            print(f"A 'push' event was triggered by {pusher_name}!")
        elif github_event == 'issues':
            issue_title = payload.get('issue', {}).get('title', 'Untitled')
            print(f"An 'issue' event was triggered for issue: {issue_title}")
        # ------------------------------------

        # Send a success response back to the service that sent the webhook.
        # It's important to send a 2xx response to let the service know
        # that you successfully received the data. Otherwise, it might try to resend it.
        return jsonify({"status": "success", "message": "Webhook received"}), 200
    else:
        # If the request is not JSON, return an error.
        return jsonify({"status": "error", "message": "Request must be JSON"}), 400

# This part runs the Flask application.
# The check `if __name__ == '__main__':` ensures this code only runs
# when the script is executed directly (not when imported as a module).
# `debug=True` enables debug mode, which provides helpful error messages.
if __name__ == '__main__':
    # Run the app on host '0.0.0.0' to make it accessible from outside the container/machine
    # and on port 5000.
    app.run(host='0.0.0.0', port=5000, debug=True)

```
# **Flashcards:**

---

What is a webhook?;;A webhook is an automated HTTP POST request sent from one application (the provider) to another (the subscriber) when a specific event occurs. It's also known as a "web callback" or a "reverse API."

What is the main difference between using a webhook and polling an API?;;Polling is a "pull" model where the client repeatedly asks the server for updates. Webhooks are a "push" model where the server automatically sends updates to the client when an event happens. Webhooks are more efficient and provide real-time data.

Briefly describe the 3 main steps of how a webhook works.;;1. Register: The client provides a URL endpoint to the server for a specific event. 2. Event Occurs: The event happens on the server. 3. Push: The server sends an HTTP POST request with a data payload (usually JSON) to the registered URL.

What is the "payload" in the context of a webhook?;;The payload is the actual data about the event that is sent in the body of the HTTP POST request. It is typically formatted as JSON and contains all relevant details of the event.

Give a common real-world example of a webhook in action.;;In a CI/CD pipeline, a service like GitHub sends a webhook to a CI server (e.g., Jenkins) when a developer pushes new code. This webhook triggers an automated build and test process.

What is a good analogy for webhooks vs. polling?;;Webhooks are like a magazine subscription: you provide your address once, and the publisher sends each new issue to you when it's ready. Polling is like going to the newsstand every day just to see if the new issue has arrived yet.