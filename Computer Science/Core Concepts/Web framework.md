---
memory: to_finish
tags:
  - will_learn
language:
  - Core Concepts
review-date:
last-reviewed: ""
scheda: done
visit-count: 0
confidence-level: 1
consecutive-correct: 0
last-struggle-date: ""
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
Web frameworks solve the fundamental problem of web development complexity and repetition. Without frameworks, developers would need to manually handle HTTP requests/responses, routing, templating, database connections, security, session management, and countless other web-specific tasks for every project. This leads to massive code duplication, security vulnerabilities, inconsistent architectures, and reinventing the wheel repeatedly.

Web frameworks are crucial because they provide standardized solutions to common web development problems, enforce best practices, improve security through battle-tested code, accelerate development through code reuse, and enable teams to focus on business logic rather than low-level web infrastructure. They're the foundation of modern web development, making it possible to build complex web applications efficiently and maintainably.

# **Core Explanation:**

---

A web framework is a software platform that provides a structured foundation for building web applications. It abstracts common web development tasks and provides reusable components, tools, and conventions that developers can use to create web applications more efficiently.

**Core Components:**

- **Routing**: Maps URLs to specific handlers or controllers
- **Request/Response Handling**: Processes HTTP requests and generates responses
- **Templating**: Generates dynamic HTML from templates and data
- **Database Integration**: ORM/ODM for database operations
- **Security**: Authentication, authorization, CSRF protection, input validation
- **Session Management**: User state persistence across requests
- **Middleware/Filters**: Cross-cutting concerns like logging, authentication
- **Static File Serving**: Handling CSS, JavaScript, images, etc.

**Framework Categories:**

**Full-Stack Frameworks** (Opinionated, "batteries included"):

- **Django (Python)**: MVT architecture, admin interface, ORM
- **Ruby on Rails (Ruby)**: MVC, Active Record, convention over configuration
- **Laravel (PHP)**: Eloquent ORM, Blade templating, Artisan CLI
- **Spring Boot (Java)**: Enterprise-grade, dependency injection, auto-configuration
- **ASP.NET Core (C

# **Cross-platform, high-performance, extensive ecosystem

**[[Microframework]]s** (Minimalist, flexible):

- **Flask (Python)**: Lightweight, extensible through blueprints
- **Express.js (Node.js)**: Minimal, middleware-based
- **Sinatra (Ruby)**: Simple DSL for web apps
- **Gin (Go)**: High-performance, minimal

**Frontend Frameworks** (Client-side):

- **React**: Component-based, virtual DOM, library ecosystem
- **Vue.js**: Progressive framework, template syntax, reactivity
- **Angular**: Full framework, TypeScript, dependency injection
- **Svelte**: Compile-time optimization, no virtual DOM

**Architecture Patterns:**

- **MVC (Model-View-Controller)**: Separates data, presentation, and logic
- **MVP (Model-View-Presenter)**: Variation of MVC with passive view
- **MVVM (Model-View-ViewModel)**: Two-way data binding, common in frontend
- **Component-Based**: Encapsulated, reusable UI components

**Key Characteristics:**

- **Convention over Configuration**: Sensible defaults reduce boilerplate
- **DRY (Don't Repeat Yourself)**: Reusable components and patterns
- **Separation of Concerns**: Clear boundaries between different aspects
- **Extensibility**: Plugin/extension systems for additional functionality
- **Community Ecosystem**: Third-party packages and integrations

# **Related Concepts:**

---
**Web Servers** - Infrastructure that serves web applications (Apache, Nginx, IIS). Frameworks run on top of web servers, while servers handle low-level HTTP protocol and static file serving.

**Application Servers** - Middleware that executes web applications (Tomcat, Gunicorn, IIS). Frameworks often include development servers but rely on application servers for production deployment.

**Content Management Systems (CMS)** - Pre-built web applications for content management (WordPress, Drupal). Built using frameworks but provide ready-to-use functionality rather than development tools.

**API Frameworks** - Specialized frameworks for building APIs (FastAPI, Swagger, GraphQL). Often lighter weight than full web frameworks, focusing on data exchange rather than HTML generation.

**ORM/ODM (Object-Relational/Document Mapping)** - Database abstraction layers often integrated into frameworks.

Examples include Django ORM, ActiveRecord, Eloquent, Mongoose.

**Template Engines** - Systems for generating dynamic HTML (Jinja2, ERB, Blade, Handlebars). Most frameworks include or integrate with template engines for view rendering.

**Middleware Pattern** - Functions that process requests before reaching handlers. Common in frameworks like Express.js, Django, and ASP.NET Core for cross-cutting concerns.

**Dependency Injection** - Design pattern for managing object dependencies, built into frameworks like Spring, Angular, and ASP.NET Core.

**Single Page Applications (SPAs)** - Web apps that load once and update dynamically. Built with frontend frameworks and communicate with backend APIs.

**Server-Side Rendering (SSR)** - Generating HTML on the server before sending to client. Frameworks like Next.js, Nuxt.js, and SvelteKit specialize in this approach.

# **Examples:**

---
```python

# Django (Python) - Full-stack framework example

# models.py - Model layer (database representation)
from django.db import models

class User(models.Model):

# Django ORM automatically creates database tables
 username = models.CharField(max_length=100, unique=True)
 email = models.EmailField
 created_at = models.DateTimeField(auto_now_add=True)

 def __str__(self):
 return self.username

# views.py - Controller layer (business logic)
from django.shortcuts import render, redirect
from django.http import JsonResponse
from .models import User

def user_list(request):

# Django handles HTTP request parsing automatically
 users = User.objects.all

# ORM query - no SQL needed

# Template rendering with context data
 return render(request, 'users/list.html', {'users': users})

def create_user(request):
 if request.method == 'POST':

# Django handles form data parsing
 username = request.POST.get('username')
 email = request.POST.get('email')

# Model validation and saving
 user = User.objects.create(username=username, email=email)
 return redirect('user_list')

# URL reversal by name

 return render(request, 'users/create.html')

def api_users(request):

# API endpoint returning JSON
 users = User.objects.values('id', 'username', 'email')
 return JsonResponse(list(users), safe=False)

# urls.py - URL routing configuration
from django.urls import path
from . import views

urlpatterns = [

# URL patterns mapped to view functions
 path('users/', views.user_list, name='user_list'),
 path('users/create/', views.create_user, name='create_user'),
 path('api/users/', views.api_users, name='api_users'),
]

# templates/users/list.html - Template layer (presentation)
<!-- Django template syntax for dynamic content -->
<h1>Users</h1>
<ul>
 {% for user in users %} <!-- Template loop -->
 <li>{{ user.username }} - {{ user.email }}</li> <!-- Variable interpolation -->
 {% empty %}
 <li>No users found</li> <!-- Conditional rendering -->
 {% endfor %}
</ul>
<a href="{% url 'create_user' %}">Create New User</a> <!-- URL generation -->
```

```javascript
// Express.js (Node.js) - Microframework example
const express = require('express');
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const app = express;

// Middleware setup - functions that process requests
app.use(express.json); // Parse JSON request bodies
app.use(express.static('public')); // Serve static files

// Database model using Mongoose ODM
const UserSchema = new mongoose.Schema({
 username: { type: String, required: true, unique: true },
 email: { type: String, required: true, unique: true },
 password: { type: String, required: true },
 createdAt: { type: Date, default: Date.now }
});

const User = mongoose.model('User', UserSchema);

// Authentication middleware - reusable across routes
const authenticateToken = (req, res, next) => {
 const authHeader = req.headers['authorization'];
 const token = authHeader && authHeader.split(' ');

 if (!token) {
 return res.status(401).json({ error: 'Access token required' });
 }

 jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
 if (err) return res.status(403).json({ error: 'Invalid token' });
 req.user = user; // Attach user to request object
 next; // Continue to next middleware/route handler
 });
};

// Route handlers - URL endpoints with business logic
app.get('/api/users', authenticateToken, async (req, res) => {
 try {
 // Database query with async/await
 const users = await User.find({}, '-password'); // Exclude password field
 res.json(users);
 } catch (error) {
 res.status(500).json({ error: 'Server error' });
 }
});

app.post('/api/users', async (req, res) => {
 try {
 const { username, email, password } = req.body;

 // Input validation
 if (!username || !email || !password) {
 return res.status(400).json({ error: 'All fields required' });
 }

 // Hash password for security
 const hashedPassword = await bcrypt.hash(password, 10);

 // Create user with ODM
 const user = new User({
 username,
 email,
 password: hashedPassword
 });

 await user.save;

 // Return user without password
 const { password: _, ...userResponse } = user.toObject;
 res.status(201).json(userResponse);

 } catch (error) {
 if (error.code === 11000) { // Duplicate key error
 res.status(400).json({ error: 'Username or email already exists' });
 } else {
 res.status(500).json({ error: 'Server error' });
 }
 }
});

app.post('/api/login', async (req, res) => {
 try {
 const { username, password } = req.body;

 // Find user in database
 const user = await User.findOne({ username });
 if (!user) {
 return res.status(400).json({ error: 'Invalid credentials' });
 }

 // Verify password
 const isMatch = await bcrypt.compare(password, user.password);
 if (!isMatch) {
 return res.status(400).json({ error: 'Invalid credentials' });
 }

 // Generate JWT token
 const token = jwt.sign(
 { userId: user._id, username: user.username },
 process.env.JWT_SECRET,
 { expiresIn: '24h' }
 );

 res.json({ token, user: { id: user._id, username: user.username } });

 } catch (error) {
 res.status(500).json({ error: 'Server error' });
 }
});

// Error handling middleware - catches all errors
app.use((error, req, res, next) => {
 console.error(error.stack);
 res.status(500).json({ error: 'Something went wrong!' });
});

// Start server
const PORT = process.env.PORT || 3000;
app.listen(PORT, => {
 console.log(`Server running on port ${PORT}`);
});
```

```jsx
// React (Frontend Framework) - Component-based architecture
import React, { useState, useEffect } from 'react';
import axios from 'axios';

// Custom hook for API calls - reusable logic
const useUsers = => {
 const [users, setUsers] = useState();
 const [loading, setLoading] = useState(false);
 const [error, setError] = useState(null);

 const fetchUsers = async => {
 setLoading(true);
 try {
 const response = await axios.get('/api/users');
 setUsers(response.data);
 setError(null);
 } catch (err) {
 setError('Failed to fetch users');
 } finally {
 setLoading(false);
 }
 };

 useEffect( => {
 fetchUsers; // Fetch on component mount
 }, );

 return { users, loading, error, refetch: fetchUsers };
};

// User component - encapsulates user display logic
const UserCard = ({ user }) => {
 return (
 <div className="user-card">
 <h3>{user.username}</h3>
 <p>{user.email}</p>
 <small>Joined: {new Date(user.createdAt).toLocaleDateString}</small>
 </div>
 );
};

// Form component - handles user creation
const CreateUserForm = ({ onUserCreated }) => {
 const [formData, setFormData] = useState({
 username: '',
 email: '',
 password: ''
 });
 const [submitting, setSubmitting] = useState(false);

 // Handle form input changes
 const handleChange = (e) => {
 setFormData({
 ...formData,
 [e.target.name]: e.target.value
 });
 };

 // Handle form submission
 const handleSubmit = async (e) => {
 e.preventDefault;
 setSubmitting(true);

 try {
 await axios.post('/api/users', formData);
 setFormData({ username: '', email: '', password: '' }); // Reset form
 onUserCreated; // Callback to refresh user list
 } catch (error) {
 console.error('Error creating user:', error);
 } finally {
 setSubmitting(false);
 }
 };

 return (
 <form onSubmit={handleSubmit} className="create-user-form">
 <h2>Create New User</h2>
 <input
 type="text"
 name="username"
 placeholder="Username"
 value={formData.username}
 onChange={handleChange}
 required
 />
 <input
 type="email"
 name="email"
 placeholder="Email"
 value={formData.email}
 onChange={handleChange}
 required
 />
 <input
 type="password"
 name="password"
 placeholder="Password"
 value={formData.password}
 onChange={handleChange}
 required
 />
 <button type="submit" disabled={submitting}>
 {submitting ? 'Creating...' : 'Create User'}
 </button>
 </form>
 );
};

// Main App component - orchestrates the application
const App = => {
 const { users, loading, error, refetch } = useUsers;

 if (loading) return <div>Loading...</div>;
 if (error) return <div>Error: {error}</div>;

 return (
 <div className="app">
 <h1>User Management</h1>

 {/* User creation form */}
 <CreateUserForm onUserCreated={refetch} />

 {/* User list */}
 <div className="users-container">
 <h2>Users ({users.length})</h2>
 {users.length === 0 ? (
 <p>No users found</p>
 ) : (
 <div className="users-grid">
 {users.map(user => (
 <UserCard key={user._id} user={user} />
 ))}
 </div>
 )}
 </div>
 </div>
 );
};

export default App;
```

```ruby

# Ruby on Rails - Convention over Configuration example

# app/models/user.rb - Model with validations
class User < ApplicationRecord

# Rails automatically maps to 'users' table
 validates :username, presence: true, uniqueness: true
 validates :email, presence: true, uniqueness: true, format: { with: URI::MailTo::EMAIL_REGEXP }
 validates :password, presence: true, length: { minimum: 6 }

 has_secure_password

# Automatically handles password hashing

# Scope for filtering - chainable query methods
 scope :recent, -> { where('created_at > ?', 1.week.ago) }
 scope :by_username, ->(username) { where(username: username) }

# Custom method
 def display_name
 "

# {username} (

# {email})"
 end
end

# app/controllers/users_controller.rb - Controller with RESTful actions
class UsersController < ApplicationController
 before_action :set_user, only: [:show, :edit, :update, :destroy]
 before_action :authenticate_user!, except: [:index, :show]

# Authentication filter

# GET /users - List all users
 def index
 @users = User.page(params[:page]).per(10)

# Pagination with Kaminari gem

# Respond to different formats
 respond_to do |format|
 format.html

# Renders app/views/users/index.html.erb
 format.json { render json: @users }
 end
 end

# GET /users/1 - Show single user
 def show
 respond_to do |format|
 format.html
 format.json { render json: @user }
 end
 end

# GET /users/new - New user form
 def new
 @user = User.new
 end

# POST /users - Create user
 def create
 @user = User.new(user_params)

 if @user.save
 redirect_to @user, notice: 'User was successfully created.'
 else
 render :new

# Re-render form with errors
 end
 end

# GET /users/1/edit - Edit user form
 def edit
 end

# PATCH/PUT /users/1 - Update user
 def update
 if @user.update(user_params)
 redirect_to @user, notice: 'User was successfully updated.'
 else
 render :edit
 end
 end

# DELETE /users/1 - Delete user
 def destroy
 @user.destroy
 redirect_to users_url, notice: 'User was successfully deleted.'
 end

 private

# Find user by ID - Rails automatically converts params[:id]
 def set_user
 @user = User.find(params[:id])
 end

# Strong parameters - security feature to prevent mass assignment
 def user_params
 params.require(:user).permit(:username, :email, :password)
 end
end

# config/routes.rb - RESTful routing
Rails.application.routes.draw do

# Creates all 7 RESTful routes automatically
 resources :users

# Custom routes
 get 'users/search/:term', to: 'users

# search'

# API namespace
 namespace :api do
 namespace :v1 do
 resources :users, only: [:index, :show, :create]
 end
 end

 root 'users

# index'

# Root route
end

# app/views/users/index.html.erb - View template with helpers
<h1>Users</h1>

<%= link_to 'New User', new_user_path, class: 'btn btn-primary' %>

<table class="table">
 <thead>
 <tr>
 <th>Username</th>
 <th>Email</th>
 <th>Created</th>
 <th colspan="3">Actions</th>
 </tr>
 </thead>
 <tbody>
 <% @users.each do |user| %>
 <tr>
 <td><%= user.username %></td>
 <td><%= user.email %></td>
 <td><%= time_ago_in_words(user.created_at) %> ago</td>
 <td><%= link_to 'Show', user %></td>
 <td><%= link_to 'Edit', edit_user_path(user) %></td>
 <td><%= link_to 'Delete', user, method: :delete,
 confirm: 'Are you sure?' %></td>
 </tr>
 <% end %>
 </tbody>
</table>

<%= paginate @users %> <!-- Pagination links -->
```

# **Flashcards:**

---
What is a web framework and what fundamental problem does it solve?;; A web framework is a software platform that provides structured foundation for building web applications, solving the problem of web development complexity by abstracting common tasks like routing, templating, database integration, and security.

What's the difference between full-stack frameworks and microframeworks?;; Full-stack frameworks (Django, Rails, Laravel) provide comprehensive, opinionated solutions with built-in features, while microframeworks (Flask, Express.js, Sinatra) offer minimal core functionality and require developers to add features as needed.

What are the core components typically found in web frameworks?;; Routing (URL mapping), request/response handling, templating (dynamic HTML generation), database integration (ORM/ODM), security features, session management, middleware/filters, and static file serving.

What is the MVC pattern and how does it relate to web frameworks?;; MVC (Model-View-Controller) separates applications into three layers: Model (data/business logic), View (presentation/UI), Controller (handles requests/coordinates between Model and View). Many web frameworks implement this pattern for organized code structure.

What is middleware in web frameworks and why is it important?;; Middleware are functions that process requests before they reach route handlers, handling cross-cutting concerns like authentication, logging, CORS, and error handling. They provide modular, reusable solutions for common web application needs.

How do frontend frameworks like React differ from backend frameworks like Django?;; Frontend frameworks run in the browser and handle user interface, state management, and user interactions, while backend frameworks run on servers and handle HTTP requests, databases, APIs, and business logic. They often work together in full-stack applications.