---
memory: to_finish
tags:
 - learned
language:
 - Core Concepts
review-date: ""
last-reviewed: 2025-07-23
scheda: done
visit-count: 5
confidence-level: 2
consecutive-correct: 3
last-struggle-date: 2025-07-03

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
Before REST (Representational State Transfer), there was no standardized architectural style for creating web services. Custom protocols or more rigid, complex standards like SOAP were common, leading to tightly coupled systems that were difficult to scale and maintain.

REST was introduced to provide a set of architectural principles for designing networked applications. It leverages the existing, proven infrastructure of the web, primarily the HTTP protocol. Its primary application is in building APIs (Application Programming Interfaces) that allow different software systems to communicate with each other over the internet in a simple, scalable, and stateless manner.

It is critically important because it provides a common design paradigm that enables:

- **Decoupling:** Separating the client and the server, allowing them to evolve independently.
- **Scalability:** The stateless nature of REST allows for easy scaling of the server, as it doesn't need to maintain client session information.
- **Simplicity & Flexibility:** By using standard HTTP methods and data formats like JSON, RESTful APIs are easy to understand, consume, and build upon across a vast range of programming languages and platforms.

# **Core Explanation:**

---
- **REST (Representational State Transfer):** This is not a protocol or a standard, but an **architectural style** for distributed systems. It defines a set of constraints that, when applied, lead to a scalable, performant, and reliable system. ==The core idea is that you are transferring a "representation" of the state of a resource (e.g., as a JSON or XML file) between the client and server.==

- **REST API:** An API that adheres to the principles and constraints of the REST architectural style.
 [[JavaScript REST APIs]]
- **RESTful API:** This term is often used interchangeably with REST API. A "RESTful" API is one that fully implements the REST constraints.


The six guiding constraints of REST are:

1. **Client-Server Architecture:** The client (which handles the user interface) and the server (which stores the data) are separated. This separation of concerns allows them to be developed and scaled independently.
2. **Stateless:** Each request from a client to the server must contain all the information needed to understand and complete the request. The server does not store any information about the client's session state between requests. This enhances scalability and reliability.
3. **Cacheable:** Responses must, implicitly or explicitly, define themselves as cacheable or non-cacheable. This allows clients or intermediaries to cache responses, which can dramatically improve performance and reduce server load.
4. **Uniform Interface:** This is a key principle that simplifies and decouples the architecture. It has four sub-constraints:
 - _Identification of resources:_ Resources (e.g., a user, a product) are identified by stable identifiers, which are URIs (e.g., `/api/users/123`).
 - _Manipulation of resources through representations:_ The client interacts with a representation of the resource (e.g., a JSON object). This representation contains the data and metadata needed to modify or delete the resource on the server.
 - _Self-descriptive messages:_ Each request is self-contained. For example, an HTTP request includes the method (GET, POST), the URI, the headers (specifying the data format like `Content-Type: application/json`), and optionally a body.
 - _Hypermedia as the Engine of Application State (HATEOAS):_ Responses from the server should include links (hypermedia) that guide the client on what actions it can take next. For example, a response for a user might include links to view their orders or update their profile.
5. **Layered System:** A client cannot ordinarily tell whether it is connected directly to the end server or to an intermediary along the2 way (like a load balancer or a cache). This allows for better scalability and security.
6. **Code on Demand (Optional):** Servers can temporarily extend or customize the functionality of a client by transferring logic that it can execute (e.g., by sending JavaScript code). This is the only optional constraint.

# **Related Concepts:**

---
- **HTTP (Hypertext Transfer Protocol):** REST is not HTTP, but it is built on it. RESTful APIs use HTTP as their communication protocol. The uniform interface of REST maps directly to HTTP's methods:
 - `GET`: Retrieve a resource.
 - `POST`: Create a new resource.
 - `PUT`: Update/replace an existing resource.
 - `DELETE`: Delete a resource.
- **API (Application Programming Interface):** An API is a general term for a set of rules and tools for building software and applications. A REST API is a specific _kind_ of API that follows the REST architectural style.
- **CRUD (Create, Read, Update, Delete):** These are the four basic operations for persistent storage. They map directly to the HTTP methods used in RESTful APIs: `POST` (Create), `GET` (Read), `PUT` (Update), and `DELETE` (Delete).
- **[[JavaScript JSON (JavaScript Object Notation)]]:** A lightweight data-interchange format. While REST is format-agnostic, JSON is the most common format used for sending and receiving data in RESTful APIs due to its human-readability and ease of parsing by machines.
- **SOAP (Simple Object Access Protocol):** An older, protocol-based alternative to REST for web services. SOAP is more rigid, has a stricter standard, and often relies on XML. REST is generally considered simpler, more flexible, and more lightweight than SOAP.

# **Examples:**

---
This JavaScript example uses the browser's `fetch` API to interact with a public RESTful API (`JSONPlaceholder`) that provides sample data.

```js
// This is an asynchronous function to allow us to use the 'await' keyword.
// This makes handling promises from the fetch API cleaner.
async function fetchUserData {
 try {
 // Define the URI (Uniform Resource Identifier) for the resource we want.
 // In this case, we are requesting the user with ID 1.
 // This is an example of the 'Identification of resources' principle.
 const userEndpoint = '.typicode.com/users/1';

 // Make a GET request to the specified endpoint using the fetch API.
 // fetch returns a Promise that resolves to the Response object.
 // This is the 'Client-Server' interaction over HTTP.
 const response = await fetch(userEndpoint);

 // RESTful APIs use HTTP status codes to indicate the outcome.
 // A status code of 200 OK means the request was successful.
 // We should always check if the response was successful before proceeding.
 if (!response.ok) {
 // If the response is not ok (e.g., 404 Not Found, 500 Internal Server Error),
 // we throw an error to be caught by the catch block.
 throw new Error(`HTTP error! status: ${response.status}`);
 }

 // The response body is a stream. We need to parse it as JSON to use it.
 // .json is an asynchronous operation that also returns a promise.
 // This is the "Representation" of the resource's state (in JSON format).
 const userData = await response.json;

 // Now we can use the data in our application.
 // The userData object is a JavaScript representation of the JSON data.
 console.log('User Data:', userData);
 console.log(`Username: ${userData.name}, Email: ${userData.email}`);

 } catch (error) {
 // If any part of the try block fails (e.g., network error, HTTP error),
 // the error is caught here and logged to the console.
 console.error('Could not fetch user data:', error);
 }
}

// Call the function to execute the API request.
fetchUserData;
```

# **Flashcards:**

---
What is REST?;;An architectural style, not a protocol, for designing networked applications. It's based on a set of constraints that promote scalability and simplicity, leveraging the web's existing HTTP protocol.

What does 'Stateless' mean in the context of REST?;;It means each request from a client to the server must contain all the information needed to be understood and processed. The server does not store any client session information between requests.

How does REST use HTTP methods?;;REST maps the basic CRUD (Create, Read, Update, Delete) operations to HTTP methods: POST (Create), GET (Read), PUT (Update), and DELETE (Delete) to manipulate resources identified by URIs.