---
memory: to_finish
tags:
  - learning
language:
  - Core Concepts
review-date: 2025-11-25
last-reviewed: 2025-10-17
scheda: done
visit-count: 2
confidence-level: 1.5
consecutive-correct: 1
last-struggle-date: 2025-10-06
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

==Microframeworks solve the problem of framework bloat and over-engineering. Traditional full-stack frameworks like Django, Ruby on Rails, or Spring Boot come with extensive built-in functionality, but many projects don't need all these features. This leads to unnecessary complexity, larger application sizes, steeper learning curves, and potential security vulnerabilities from unused components.==

Microframeworks address the =="goldilocks problem" of web development - providing just enough structure to be productive without imposing unnecessary constraints or overhead==. They're crucial for rapid prototyping, microservices architecture, APIs, and situations where you need fine-grained control over your application's components. They embody the Unix philosophy of "do one thing and do it well."

# **Core Explanation:**
---

A microframework is a minimalist web framework that provides<mark style="background: #BBFABBA6;"> only the essential components needed to build web applications, typically including routing, request/response handling, and basic templating. Unlike full-stack frameworks, microframeworks start with a minimal core and allow developers to add only the components they need.</mark>

**Key Characteristics:**

- **Minimal Core**: Provides <mark style="background: #D2B3FFA6;">only essential web functionality (routing, HTTP handling)</mark>
- **Modular Design**: <mark style="background: #D2B3FFA6;">Features are added through extensions or third-party libraries</mark>
- **Lightweight**:<mark style="background: #D2B3FFA6;"> Small codebase </mark>with minimal dependencies
- **Flexibility**: <mark style="background: #D2B3FFA6;">Fewer conventions and assumptions</mark> about application structure
- **Extensible**: <mark style="background: #D2B3FFA6;">Easy to add functionality</mark> through plugins or middleware

**Popular Microframeworks:**
- **[[Flask (web framework)]] ([[Python]])** - The most well-known microframework, providing routing, templating (Jinja2), and request handling. Extensions available for databases, authentication, etc.
- **[[JavaScript Express.js]] ([[Node.js - Server-Side JavaScript]])** - Minimal and flexible web framework providing middleware system, routing, and HTTP utilities.
- **Sinatra (Ruby)** - Pioneered the microframework approach with its simple DSL for web applications.
- **Fastify (Node.js)** - High-performance alternative to Express with built-in JSON schema validation.
- **Gin (Go)** - Lightweight HTTP web framework with middleware support and JSON validation.
- **Fiber (Go)** - Express-inspired framework built on top of Fastify's HTTP engine.
- **Actix-web (Rust)** - High-performance web framework with actor-based architecture.
- **Warp (Rust)** - Composable web framework focusing on filters and type safety.

**Architecture Pattern**: ==Microframeworks typically follow a request-response cycle with middleware/filter chains for cross-cutting concerns like authentication, logging, and error handling.==

# **Related Concepts:**
---

**Full-Stack Frameworks** - The opposite of microframeworks.

Examples include Django, Rails, Laravel, and Spring Boot. These provide comprehensive solutions but with more complexity and opinions about application structure.

**Middleware Pattern** - Most microframeworks use middleware to handle cross-cutting concerns. Middleware functions execute in sequence, allowing modular handling of authentication, logging, CORS, etc.

**Microservices Architecture** - Microframeworks are often used to build microservices due to their lightweight nature and ability to create focused, single-responsibility services.

**API-First Development** - Microframeworks excel at building APIs and RESTful services, making them popular for backend services that serve frontend applications or mobile apps.

**Convention over Configuration** - While full-stack frameworks embrace this principle, microframeworks typically favor explicit configuration, giving developers more control but requiring more setup.

**Dependency Injection** - Some microframeworks provide DI containers, while others rely on manual dependency management or third-party solutions.

**Template Engines** - Microframeworks often integrate with lightweight template engines (Jinja2, Mustache, Handlebars) rather than providing their own comprehensive templating systems.

# **Examples:**
---

```python

# Flask (Python) - Classic microframework example
from flask import Flask, jsonify, request

# Create Flask application instance - minimal setup required
app = Flask(__name__)

# Simple route decorator - core microframework feature
@app.route('/')
def home:
 return "Hello, World!"

# Basic response

# Route with URL parameters - flexible routing
@app.route('/users/<int:user_id>')
def get_user(user_id):

# Simulate database lookup
 user = {"id": user_id, "name": f"User {user_id}"}
 return jsonify(user)

# JSON response with minimal setup

# HTTP method handling - RESTful API support
@app.route('/users', methods=['POST'])
def create_user:

# Access request data - built-in request handling
 data = request.get_json

# Minimal validation (in real app, use extensions)
 if not data or 'name' not in data:
 return jsonify({"error": "Name required"}), 400

# Simulate user creation
 new_user = {"id": 123, "name": data['name']}
 return jsonify(new_user), 201

# Middleware example - before_request decorator
@app.before_request
def before_request:

# Log every request - cross-cutting concern
 print(f"Request: {request.method} {request.path}")

if __name__ == '__main__':
 app.run(debug=True)

# Development server - minimal configuration
```

```javascript
// Express.js (Node.js) - Microframework with middleware pattern
const express = require('express');
const app = express;

// Built-in middleware for JSON parsing
app.use(express.json);

// Custom middleware function - demonstrates modularity
const requestLogger = (req, res, next) => {
 console.log(`${req.method} ${req.path} - ${new Date.toISOString}`);
 next; // Pass control to next middleware
};

// Apply middleware globally
app.use(requestLogger);

// Simple route handler
app.get('/', (req, res) => {
 res.json({ message: 'Hello from Express!' });
});

// Route with parameters and query strings
app.get('/api/users/:id', (req, res) => {
 const userId = req.params.id; // URL parameter
 const include = req.query.include; // Query string parameter

 // Simulate user lookup
 const user = {
 id: userId,
 name: `User ${userId}`,
 email: `user${userId}@example.com`
 };

 // Conditional response based on query parameter
 if (include === 'profile') {
 user.profile = { bio: 'A sample user profile' };
 }

 res.json(user);
});

// Error handling middleware - placed after routes
app.use((err, req, res, next) => {
 console.error(err.stack);
 res.status(500).json({ error: 'Something went wrong!' });
});

// Start server with minimal configuration
const PORT = process.env.PORT || 3000;
app.listen(PORT, => {
 console.log(`Server running on port ${PORT}`);
});
```

```go
// Gin (Go) - High-performance microframework
package main

import (
 "net/http"
 "github.com/gin-gonic/gin"
)

// User struct for JSON serialization
type User struct {
 ID int `json:"id"`
 Name string `json:"name"`
}

func main {
 // Create Gin router with default middleware (logger, recovery)
 r := gin.Default

 // Custom middleware for CORS - shows extensibility
 r.Use(func(c *gin.Context) {
 c.Header("Access-Control-Allow-Origin", "*")
 c.Next // Continue to next handler
 })

 // Simple GET route
 r.GET("/", func(c *gin.Context) {
 c.JSON(http.StatusOK, gin.H{
 "message": "Hello from Gin!",
 })
 })

 // Route group for API versioning - organizational feature
 v1 := r.Group("/api/v1")
 {
 // GET route with path parameter
 v1.GET("/users/:id", func(c *gin.Context) {
 id := c.Param("id") // Extract path parameter

 // Simulate user lookup
 user := User{
 ID: 1,
 Name: "John Doe",
 }

 c.JSON(http.StatusOK, user)
 })

 // POST route with JSON binding
 v1.POST("/users", func(c *gin.Context) {
 var newUser User

 // Bind JSON request body to struct - built-in validation
 if err := c.ShouldBindJSON(&newUser); err != nil {
 c.JSON(http.StatusBadRequest, gin.H{"error": err.Error})
 return
 }

 // Simulate user creation
 newUser.ID = 123
 c.JSON(http.StatusCreated, newUser)
 })
 }

 // Start server - minimal configuration
 r.Run(":8080") // Listen on port 8080
}
```

```rust
// Warp (Rust) - Composable microframework with filters
use warp::Filter;
use serde::{Deserialize, Serialize};

# [derive(Serialize, Deserialize)]
struct User {
 id: u32,
 name: String,
}

# [tokio::main]
async fn main {
 // CORS filter - reusable component
 let cors = warp::cors
 .allow_any_origin
 .allow_headers(vec!["content-type"])
 .allow_methods(vec!["GET", "POST"]);

 // GET route filter - composable routing
 let get_users = warp::path("users")
 .and(warp::get)
 .map(|| {
 // Simulate user list
 let users = vec![
 User { id: 1, name: "Alice".to_string },
 User { id: 2, name: "Bob".to_string },
 ];
 warp::reply::json(&users)
 });

 // POST route with JSON body parsing
 let create_user = warp::path("users")
 .and(warp::post)
 .and(warp::body::json) // Parse JSON body
 .map(|mut user: User| {
 // Simulate user creation
 user.id = 123;
 warp::reply::json(&user)
 });

 // Combine routes with OR - filter composition
 let routes = get_users
 .or(create_user)
 .with(cors); // Apply CORS to all routes

 // Start server - minimal setup
 warp::serve(routes)
 .run(([127, 0, 0, 1], 3030))
 .await;
}
```

# **Flashcards:**
---

What is a microframework?;; A minimalist web framework that provides only essential components (routing, request/response handling) and allows developers to add only the features they need through extensions or third-party libraries.

What problem do microframeworks solve?;; They solve framework bloat and over-engineering by providing just enough structure to be productive without unnecessary complexity, larger sizes, or unused features that come with full-stack frameworks.

Name three popular microframeworks and their languages;; Flask (Python), Express.js (Node.js), and Sinatra (Ruby). Other examples include Gin (Go), Fastify (Node.js), and Warp (Rust).

How do microframeworks differ from full-stack frameworks?;; Microframeworks start with a minimal core and require explicit addition of features, while full-stack frameworks come with comprehensive built-in functionality and follow "convention over configuration."

What is the middleware pattern in microframeworks?;; A pattern where functions execute in sequence during request processing, allowing modular handling of cross-cutting concerns like authentication, logging, CORS, and error handling.

When should you choose a microframework over a full-stack framework?;; When building APIs, microservices, rapid prototypes, or when you need fine-grained control over application components and want to avoid unnecessary complexity or overhead.