---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-11-07
scheda: done
visit-count: 7
confidence-level: 3
consecutive-correct: 4
last-struggle-date: 2025-08-06
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
HTTP methods solve the fundamental problem of defining <mark style="background: #FFF3A3A6;">standardized ways for clients and servers to communicate over the web by specifying the type of action being requested.</mark> They provide a uniform interface for different operations on web resources, enabling clients to clearly indicate whether they want to retrieve data, submit new information, update existing content, or delete resources. This standardization is crucial for web development, API design, and building scalable web applications because it creates predictable behavior patterns that both humans and machines can understand and implement consistently across different systems and platforms.

# **Core Explanation:**
---
HTTP methods (also called HTTP verbs) are standardized request types that indicate the desired <mark style="background: #FFB86CA6;">action to be performed on a specified resource.</mark> They form part of the HTTP protocol and define the semantics of client-server communication on the web.

The most commonly used HTTP methods include:

>**GET**: Retrieves data from the server without causing any side effects. It should be ==idempotent== <mark style="background: #FF5582A6;">(multiple identical requests have the same effect)</mark> and ==safe== <mark style="background: #FF5582A6;">(doesn't modify server state)</mark>.
>
>**POST**: Submits data to the server to create new resources or trigger processing. It's neither safe nor idempotent, meaning repeated requests may cause different effects.
>
>**PUT**: <mark style="background: #BBFABBA6;">Updates or creates a resource at a specific location.</mark> It's idempotent but not safe, as it modifies server state.
>
>**DELETE**: Removes a specified resource from the server. It's idempotent but not safe.
>
>**PATCH**: <mark style="background: #ADCCFFA6;">Applies partial updates to a resource, modifying only specified fields rather than replacing the entire resource.</mark>
>
>**HEAD**: <mark style="background: #D2B3FFA6;">Retrieves only the headers of a response without the body. It's identical to GET but doesn't return the response body.</mark> It's both idempotent and safe.
>
>**OPTIONS**: <mark style="background: #FFF3A3A6;">Retrieves the HTTP methods supported by the server for a specific URL</mark>. Used to determine the communication options available for a resource. It's both idempotent and safe.
>
>**CONNECT**:<mark style="background: #BBFABBA6;"> Establishes a tunnel to the server identified by the target resource.</mark> Primarily used for SSL tunneling through proxies. It's neither idempotent nor safe.
>
>**TRACE**: <mark style="background: #FF5582A6;">Performs a message loop-back test along the path to the target resource</mark>. It echoes back the received request so the client can see what servers are adding or changing in the request. It's idempotent and safe.

| Method  | Description                                            | Idempotent | Safe |
| ------- | ------------------------------------------------------ | ---------- | ---- |
| GET     | Retrieves data from the server without side effects    | Yes        | Yes  |
| HEAD    | Retrieves only headers without the response body       | Yes        | Yes  |
| POST    | Submits data to create resources or trigger processing | No         | No   |
| PUT     | Updates or creates a resource at a specific location   | Yes        | No   |
| DELETE  | Removes a specified resource from the server           | Yes        | No   |
| PATCH   | Applies partial updates to a resource                  | ==No==     | No   |
| OPTIONS | Retrieves HTTP methods supported by the server         | Yes        | Yes  |
| CONNECT | Establishes a tunnel to the server                     | ==No==     | No   |
| TRACE   | Performs a message loop-back test                      | Yes        | Yes  |
Key characteristics include:
- **Safety**: Methods that don't modify server state (GET, HEAD, OPTIONS)
- **Idempotency**: Methods that produce the same result when called multiple times (GET, PUT, DELETE)
- **Cacheable**: Some methods allow responses to be cached (primarily GET and HEAD)

Cacheable HTTP methods (primarily GET and HEAD) allow responses to be stored temporarily in a [[Cache]], enabling faster retrieval for identical future requests without hitting the server again. This improves performance and reduces server load.

==HTTP methods work by being included in the request line of an HTTP message, followed by the target URI and [[HTTP Versions]]==. The server interprets the method to determine how to process the request and what type of response to return.

# **Related Concepts:**
---
**REST (Representational State Transfer)**: An architectural style that uses HTTP methods as the uniform interface for resource manipulation, mapping CRUD operations to specific HTTP methods (GET for read, POST for create, PUT/PATCH for update, DELETE for delete).

**HTTP Status Codes**: Numeric codes returned by servers to indicate the result of processing HTTP method requests (200 for successful GET, 201 for successful POST, 404 for resource not found, etc.).

**CRUD Operations**: The four basic database operations (Create, Read, Update, Delete) that directly correspond to HTTP methods (POST/Create, GET/Read, PUT-PATCH/Update, DELETE/Delete).

**[[HTTP Headers]]**: Additional metadata sent with HTTP requests and responses that can modify method behavior, such as Content-Type for POST/PUT requests or Cache-Control for GET requests.

**URI (Uniform Resource Identifier)**: The target address that HTTP methods operate on, defining what resource the method should affect.
--> [[URI - Similarities and Differences with URL]]

**Web APIs and Endpoints**: Specific URLs that accept certain HTTP methods to perform defined operations, forming the interface for web services and RESTful APIs.

**Request/Response Cycle**: The complete communication pattern where a client sends an HTTP request with a specific method and receives an appropriate HTTP response from the server.

# **Examples:**
---
```javascript
// JavaScript Fetch API Examples - HTTP Methods in Practice
// This demonstrates how different HTTP methods are used in web applications

// GET Method - Retrieving data from server
// Safe and idempotent operation - doesn't modify server state
async function getUserData(userId) {
    try {
        // GET request to retrieve user information
        // No request body needed - data passed via URL parameters
        const response = await fetch(`/api/users/${userId}`, {
            method: 'GET',           // Explicitly specify GET method
            headers: {
                'Accept': 'application/json'  // Tell server we expect JSON
            }
        });
        
        // Check if request was successful
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const userData = await response.json();
        return userData;
    } catch (error) {
        console.error('GET request failed:', error);
    }
}

// POST Method - Creating new resources
// Not safe or idempotent - creates new data on server
async function createUser(userData) {
    try {
        // POST request to create a new user
        // Includes request body with user data
        const response = await fetch('/api/users', {
            method: 'POST',          // POST method for creating resources
            headers: {
                'Content-Type': 'application/json',  // Specify data format
                'Accept': 'application/json'
            },
            body: JSON.stringify(userData)  // Convert object to JSON string
        });
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const newUser = await response.json();
        console.log('User created successfully:', newUser);
        return newUser;
    } catch (error) {
        console.error('POST request failed:', error);
    }
}

// PUT Method - Complete resource replacement
// Idempotent but not safe - replaces entire resource
async function updateUser(userId, completeUserData) {
    try {
        // PUT request replaces the entire user resource
        // Multiple identical PUT requests have same effect (idempotent)
        const response = await fetch(`/api/users/${userId}`, {
            method: 'PUT',           // PUT method for complete updates
            headers: {
                'Content-Type': 'application/json',
                'Accept': 'application/json'
            },
            body: JSON.stringify(completeUserData)  // Complete user object
        });
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const updatedUser = await response.json();
        return updatedUser;
    } catch (error) {
        console.error('PUT request failed:', error);
    }
}

// PATCH Method - Partial resource updates
// More efficient than PUT when only updating specific fields
async function updateUserEmail(userId, newEmail) {
    try {
        // PATCH request updates only specified fields
        // Sends only the data that needs to be changed
        const response = await fetch(`/api/users/${userId}`, {
            method: 'PATCH',         // PATCH method for partial updates
            headers: {
                'Content-Type': 'application/json',
                'Accept': 'application/json'
            },
            body: JSON.stringify({ email: newEmail })  // Only email field
        });
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const updatedUser = await response.json();
        return updatedUser;
    } catch (error) {
        console.error('PATCH request failed:', error);
    }
}

// DELETE Method - Removing resources
// Idempotent but not safe - removes data from server
async function deleteUser(userId) {
    try {
        // DELETE request to remove user resource
        // No request body needed - target specified in URL
        const response = await fetch(`/api/users/${userId}`, {
            method: 'DELETE',        // DELETE method for resource removal
            headers: {
                'Accept': 'application/json'
            }
        });
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        console.log(`User ${userId} deleted successfully`);
        return true;
    } catch (error) {
        console.error('DELETE request failed:', error);
        return false;
    }
}
````

```python
# Python Flask Server Example - Handling Different HTTP Methods
# This shows how a server processes different HTTP methods

from flask import Flask, request, jsonify
import json

app = Flask(__name__)

# In-memory storage for demonstration
users = {
    1: {"id": 1, "name": "John Doe", "email": "john@example.com"},
    2: {"id": 2, "name": "Jane Smith", "email": "jane@example.com"}
}
next_user_id = 3

# GET Method Handler - Retrieve user data
# Safe operation - doesn't modify server state
@app.route('/api/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    """
    Handle GET requests to retrieve user information
    URL: GET /api/users/1
    """
    # Check if user exists in our data store
    if user_id in users:
        # Return user data with 200 OK status
        return jsonify(users[user_id]), 200
    else:
        # Return error with 404 Not Found status
        return jsonify({"error": "User not found"}), 404

# POST Method Handler - Create new user
# Not safe - modifies server state by adding new data
@app.route('/api/users', methods=['POST'])
def create_user():
    """
    Handle POST requests to create new users
    URL: POST /api/users
    Body: {"name": "New User", "email": "user@example.com"}
    """
    global next_user_id
    
    # Extract JSON data from request body
    user_data = request.get_json()
    
    # Validate required fields
    if not user_data or 'name' not in user_data or 'email' not in user_data:
        return jsonify({"error": "Name and email are required"}), 400
    
    # Create new user with auto-generated ID
    new_user = {
        "id": next_user_id,
        "name": user_data['name'],
        "email": user_data['email']
    }
    
    # Add to storage and increment ID counter
    users[next_user_id] = new_user
    next_user_id += 1
    
    # Return created user with 201 Created status
    return jsonify(new_user), 201

# PUT Method Handler - Complete resource replacement
# Idempotent - multiple identical requests have same effect
@app.route('/api/users/<int:user_id>', methods=['PUT'])
def update_user(user_id):
    """
    Handle PUT requests to completely replace user data
    URL: PUT /api/users/1
    Body: {"name": "Updated Name", "email": "updated@example.com"}
    """
    user_data = request.get_json()
    
    # Validate request body
    if not user_data or 'name' not in user_data or 'email' not in user_data:
        return jsonify({"error": "Name and email are required"}), 400
    
    # Replace entire user record (or create if doesn't exist)
    updated_user = {
        "id": user_id,
        "name": user_data['name'],
        "email": user_data['email']
    }
    
    users[user_id] = updated_user
    return jsonify(updated_user), 200

# PATCH Method Handler - Partial updates
# Updates only specified fields, leaving others unchanged
@app.route('/api/users/<int:user_id>', methods=['PATCH'])
def patch_user(user_id):
    """
    Handle PATCH requests to partially update user data
    URL: PATCH /api/users/1
    Body: {"email": "newemail@example.com"}  # Only email updated
    """
    # Check if user exists
    if user_id not in users:
        return jsonify({"error": "User not found"}), 404
    
    user_data = request.get_json()
    if not user_data:
        return jsonify({"error": "No data provided"}), 400
    
    # Update only provided fields
    user = users[user_id]
    if 'name' in user_data:
        user['name'] = user_data['name']
    if 'email' in user_data:
        user['email'] = user_data['email']
    
    return jsonify(user), 200

# DELETE Method Handler - Remove resource
# Idempotent - deleting same resource multiple times has same effect
@app.route('/api/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    """
    Handle DELETE requests to remove user
    URL: DELETE /api/users/1
    """
    # Check if user exists
    if user_id in users:
        # Remove user from storage
        deleted_user = users.pop(user_id)
        return jsonify({"message": f"User {user_id} deleted successfully"}), 200
    else:
        # Return success even if user doesn't exist (idempotent behavior)
        return jsonify({"message": "User not found or already deleted"}), 200

if __name__ == '__main__':
    app.run(debug=True)
```

```bash
# cURL Examples - HTTP Methods from Command Line
# These demonstrate how HTTP methods work at the protocol level

# GET Request - Retrieve user data
# Safe and cacheable operation
curl -X GET \
  http://localhost:5000/api/users/1 \
  -H "Accept: application/json"

# Expected Response: {"id": 1, "name": "John Doe", "email": "john@example.com"}

# POST Request - Create new user
# Includes request body with user data
curl -X POST \
  http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"name": "Alice Johnson", "email": "alice@example.com"}'

# Expected Response: {"id": 3, "name": "Alice Johnson", "email": "alice@example.com"}

# PUT Request - Complete resource replacement
# Replaces entire user record
curl -X PUT \
  http://localhost:5000/api/users/1 \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"name": "John Updated", "email": "johnupdated@example.com"}'

# PATCH Request - Partial update
# Updates only specified fields
curl -X PATCH \
  http://localhost:5000/api/users/1 \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"email": "johnpatched@example.com"}'

# DELETE Request - Remove resource
# No request body needed
curl -X DELETE \
  http://localhost:5000/api/users/1 \
  -H "Accept: application/json"

# Expected Response: {"message": "User 1 deleted successfully"}
```

# **Flashcards:**

---

What are the key differences between PUT and PATCH HTTP methods?;; PUT replaces the entire resource with the provided data and requires a complete representation, while PATCH applies partial updates to specific fields only. Both are idempotent, but PUT may create a resource if it doesn't exist, whereas PATCH typically only updates existing resources.

Which HTTP methods are considered safe and idempotent, and what do these properties mean?;; Safe methods (GET, HEAD, OPTIONS) don't modify server state. Idempotent methods (GET, PUT, DELETE, HEAD, OPTIONS) produce the same result when called multiple times. POST and PATCH are neither safe nor idempotent, meaning they can modify server state and repeated calls may have different effects.

How do HTTP methods map to CRUD operations in RESTful API design?;; CREATE maps to POST (create new resources), READ maps to GET (retrieve data), UPDATE maps to PUT (complete replacement) or PATCH (partial update), and DELETE maps to DELETE (remove resources). This mapping provides a standardized interface for resource manipulation in web APIs.