---
memory: to_finish
tags:
 - learned
language:
 - Core Concepts
review-date: ""
last-reviewed: 2025-08-12
scheda: done
visit-count: 5
confidence-level: 2
consecutive-correct: 3
last-struggle-date: 2025-07-01
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

# **Core Explanation:**

---
An API Endpoint is a ==specific URL or URI within an API== (Application Programming Interface) that allows clients (like web or mobile apps) to interact with a server. ==Each endpoint represents a specific function or resource, such as retrieving user data or submitting a form==. Endpoints are accessed using [[HTTP Methods]] (GET, POST, PUT, DELETE, etc.), and they define how clients can communicate with the backend.

Endpoints are fundamental to [[RESTful API]]s, where each endpoint corresponds to a resource (like /users or /products) and supports standard operations.

# **Related Concepts:**

---
- REST (Representational State Transfer): An architectural style for designing networked applications, often using endpoints.

- HTTP Methods: Actions like GET (retrieve), POST (create), PUT (update), DELETE (remove) used with endpoints.

- Routes: The path part of an endpoint (e.g., /api/users).

- Request/Response: The data sent to (request) and received from (response) an endpoint.

- Status Codes: HTTP codes (e.g., 200, 404, 500) indicating the result of an endpoint call.

# **Examples:**

---
#

# **Example 1 (InnoBee api.py):**

```python
from flask import Flask, request, jsonify
from werkzeug.security import generate_password_hash, check_password_hash
from datetime import datetime, timedelta
import jwt
import mongoDB

# Assuming 'mongoDB.py' contains a function to connect to MongoDB
from auth_utils import token_required

# Assuming 'auth_utils.py' contains the token_required decorator
from dotenv import load_dotenv

# Used to load environment variables from a .env file
import os

# Used to interact with the operating system environment

# Load environment variables from .env file.

# This is crucial for securely storing sensitive information like the secret key.
load_dotenv

# Define the Flask app instance.
app = Flask(__name__)

# Set the secret key for the Flask application from environment variables.

# This key is essential for securely encoding and decoding JWTs.
app.config['SECRET_KEY'] = os.getenv('SECRET_KEY')

# Connect to the MongoDB database.

# 'mongoDB.get_database' is a hypothetical function that establishes the connection.
db = mongoDB.get_database

# Get a reference to the 'users' collection within the connected database.
users_collection = db['users']

# Register endpoint: Allows new users to create an account.

# This endpoint listens for POST requests to the '/register' URL.
@app.route('/register', methods=['POST'])
def register_user:

# Parse the incoming JSON request body.

# This expects the client to send data in JSON format (e.g., {"name": "...", "email": "...", "password": "..."}).
 data = request.get_json

# Extract user registration details from the parsed JSON data.
 name = data.get('name')
 email = data.get('email')
 password = data.get('password')

# Validate input: Check if all required fields are provided.
 if not name or not email or not password:

# If any field is missing, return an error message with a 400 Bad Request status.
 return jsonify({"error": "All fields are required"}), 400

# Check if a user with the provided email already exists in the database.
 if users_collection.find_one({"email": email}):

# If the email is already registered, return an error with a 409 Conflict status.
 return jsonify({"error": "Email already registered"}), 409

# Hash the user's password before storing it in the database.

# 'generate_password_hash' uses a strong hashing algorithm (pbkdf2:sha256) for security.
 hashed_password = generate_password_hash(password, method='pbkdf2:sha256')

# Prepare the user data dictionary to be stored in MongoDB.
 user_data = {
 "name": name,
 "email": email,
 "password": hashed_password,
 "created_at": datetime.utcnow

# Record the creation timestamp in UTC.
 }

# Insert the new user document into the 'users' collection.
 result = users_collection.insert_one(user_data)

# Create a JSON Web Token (JWT) for the newly registered user.

# This token will be used for subsequent authenticated requests.
 token = jwt.encode({
 'user_id': str(result.inserted_id),

# Include the user's MongoDB ObjectId as a string.
 'exp': datetime.utcnow + timedelta(days=1)

# Set token expiration to 1 day from now.
 }, app.config['SECRET_KEY'], algorithm='HS256')

# Encode using the app's secret key and HS256 algorithm.

# Return a success message, the user's ID, and the generated JWT.

# A 201 Created status code indicates successful resource creation.
 return jsonify({
 "message": "User registered successfully!",
 "user_id": str(result.inserted_id),
 "token": token
 }), 201

# Login endpoint: Allows existing users to authenticate and receive a JWT.

# This endpoint listens for POST requests to the '/login' URL.
@app.route('/login', methods=['POST'])
def login_user:

# Parse the incoming JSON request body, expecting email and password.
 data = request.get_json

 email = data.get('email')
 password = data.get('password')

# Validate input: Ensure both email and password are provided.
 if not email or not password:
 return jsonify({"error": "Both email and password are required"}), 400

# Find the user in the database based on the provided email.
 user = users_collection.find_one({"email": email})

# If no user is found with that email, return an authentication error.
 if not user:
 return jsonify({"error": "Invalid email or password"}), 401

# Check if the provided plain-text password matches the hashed password in the database.

# 'check_password_hash' safely compares the two.
 if not check_password_hash(user['password'], password):
 return jsonify({"error": "Invalid email or password"}), 401

# If authentication is successful, create a new JWT for the user.
 token = jwt.encode({
 'user_id': str(user['_id']),

# Include the user's MongoDB ObjectId.
 'exp': datetime.utcnow + timedelta(days=1)

# Set token expiration.
 }, app.config['SECRET_KEY'], algorithm='HS256')

# Return a success message and the generated JWT.
 return jsonify({
 "message": "Login successful",
 "token": token
 })

# Example protected route: This endpoint requires a valid JWT for access.

# This endpoint listens for GET requests to the '/protected' URL.

# The `@token_required` decorator (from 'auth_utils.py') ensures that a valid token

# is present in the request headers before the `protected_route` function is executed.

# It also passes the `current_user` object (decoded from the token) to the function.
@app.route('/protected', methods=['GET'])
@token_required(app.config['SECRET_KEY'])

# Pass the secret key to the decorator for token verification.
def protected_route(current_user):

# This code only runs if the token is valid and the user is authenticated.

# It returns a personalized welcome message using data from the 'current_user' object.
 return jsonify({
 "message": f"Welcome {current_user['name']}! This is a protected route."
 })

# To run the Flask application:

# if __name__ == '__main__':

#

# app.run(debug=True) enables debug mode, which provides detailed error messages

#

# and automatically reloads the server on code changes.

# app.run(debug=True)
```
In the provided Flask example, the API Endpoints are:

1. **`/register`**
2. **`/login`**
3. **`/protected`**

Each of these strings (like `'/register'`) represents a specific URL path that clients can send requests to, and each is associated with a Python function (`register_user`, `login_user`, `protected_route`) that defines how the server will handle those requests. In the provided Flask example, the API endpoints are defined by the `@app.route` decorators. Each of these decorated functions represents a specific API endpoint that a client can interact with.

Here are the API Endpoints in your example:

1. **`/register` (POST method)**

 - This endpoint allows a client to send a POST request to create a new user account.
 - It expects user `name`, `email`, and `password` in the request body.
2. **`/login` (POST method)**

 - This endpoint allows a client to send a POST request to authenticate a user.
 - It expects user `email` and `password` in the request body. Upon successful authentication, it returns a JWT.
3. **`/protected` (GET method)**

 - This endpoint allows a client to send a GET request to access a protected resource.
 - It requires a valid JWT to be sent in the request headers for authorization. If the token is valid, it returns a personalized message.

In summary, an API endpoint is the combination of a **URL path** and an **HTTP method**. These define a specific "address" and "action" for interacting with your API.

#

# **Example 2:**

```python

# Example: Simple Flask API with endpoints

from flask import Flask, jsonify, request

app = Flask(__name__)

# This is an endpoint for retrieving all users
@app.route('/api/users', methods=['GET'])
def get_users:
 """
 GET endpoint to retrieve a list of users.
 - URL: /api/users
 - Method: GET
 - Returns: JSON list of users
 """
 users = [{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}]
 return jsonify(users)

# This is an endpoint for creating a new user
@app.route('/api/users', methods=['POST'])
def create_user:
 """
 POST endpoint to create a new user.
 - URL: /api/users
 - Method: POST
 - Expects: JSON with user data in request body
 - Returns: The created user with an ID
 """
 data = request.get_json
 new_user = {"id": 3, "name": data["name"]}
 return jsonify(new_user), 201

# This is an endpoint for retrieving a specific user by ID
@app.route('/api/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
 """
 GET endpoint to retrieve a user by ID.
 - URL: /api/users/<user_id>
 - Method: GET
 - Returns: JSON with user data or 404 if not found
 """
 users = {1: "Alice", 2: "Bob"}
 if user_id in users:
 return jsonify({"id": user_id, "name": users[user_id]})
 else:
 return jsonify({"error": "User not found"}), 404

# To run the app:

# if __name__ == '__main__':

# app.run(debug=True)
```

# **Flashcards:**

---
What is an API Endpoint?;; A specific URL or URI within an API that allows clients to interact with a server, representing a specific function or resource.
<!--SR:!2025-06-15,1,210-->

What are common HTTP methods used with API Endpoints?;; GET (retrieve), POST (create), PUT (update), DELETE (remove).

In a RESTful API, what does each endpoint typically correspond to?;; A resource (e.g., `/users`, `/products`).
<!--SR:!2025-06-17,3,250-->