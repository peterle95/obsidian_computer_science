---
memory: to_finish
tags:
 - mastered
language:
 - Core Concepts
review-date: ""
last-reviewed: 2025-08-24
scheda: done
visit-count: 1
confidence-level: 1
consecutive-correct: 1

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
# **Core Explanation:**

---
API Integration (Frontend-Backend) is the process of enabling your web frontend (like your InnoBee React app) to communicate with your backend server (like your InnoBee Flask API) to exchange data and perform operations. This communication typically happens ==over HTTP using RESTful principles==.

<mark style="background:

# ABF7F7A6;">The frontend sends **requests** to specific **API Endpoints** on the backend, and the backend processes these requests, interacts with the database, and sends back **responses** (often in JSON format).</mark>

Our journey with the InnoBee project highlighted several common challenges in this integration:

# Step-by-Step API Integration & Troubleshooting (InnoBee Project)

1. **Gaining Backend Access:**

 - Initially, you only had frontend access. For effective API integration, **backend access is absolutely necessary**. It allows you to understand data structures, validation rules, error handling, and debug issues efficiently.
 - You successfully communicated this need by emphasizing efficiency and time-saving benefits, framing it as an optimization rather than a blocker.
2. **Setting up a Multi-Root Workspace:**

 - To work with both the `InnoBee` (frontend) and `InnoBee-Backend` repositories simultaneously, a **multi-root workspace** was set up in Cursor (or VS Code).
 - This was achieved via `File -> Add Folder to Workspace`, adding both repo folders, and then `File -> Save Workspace As...` to create a `.code-workspace` file.
 - **Advantages:** This allows for cross-repo search, better IntelliSense, unified file exploration, and provides Cursor's AI with context from both codebases, which is crucial for integration tasks.
3. **Running the Backend Server:**

 - **Initial Issue: Port Conflict (8000 in use)**
 - Your `run.py` script attempted to start Flask on port 8000, but `docker-pr` (Docker process) was already using it.
 - **Solution:** You identified the process with `sudo lsof -i :8000` and eventually learned to run the backend on a different port by setting the `FLASK_PORT` environment variable: `FLASK_PORT=8001 python3 run.py`.
 - **Key Learning:** Ensure your Flask application is configured to read the `PORT` or `FLASK_PORT` environment variable to allow dynamic port assignment instead of hardcoding.
 - **Secondary Issue: Database Connection String & Multiple Initializations**
 - The backend repeatedly printed `connection_string enter-Your_Db-Url`, indicating the database connection string was a placeholder and the app was initializing multiple times.
 - **Action:** This requires you to edit your `.env` file or configuration (e.g., `config.py`) to provide a valid MongoDB connection URL. The multiple initializations might be due to Flask's reloader or an import loop, requiring further investigation if it causes stability issues, but didn't block initial server startup.
4. **Running the Frontend Server:**

 - **Initial Issue: React/Vite Syntax Error**
 - `npm start` failed due to an "Unexpected closing fragment tag does not match opening 'div' tag" in `src/pages/MySubmission/MySubmission.tsx`.
 - **Solution:** This was a basic React JSX syntax error, requiring `</div>` instead of `</>` to correctly close the `div` tag. **Always fix syntax errors first!**
 - **Successful Startup:** Once the syntax error was resolved, the frontend started on `.
5. **Connecting Frontend to Backend (CORS Error):**

 - **Issue:** After the frontend started, it showed a **CORS error**: "Access to XMLHttpRequest at '.0.1:8001/api/v1/auth/login' from origin '.0.1:3000' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource."
 - **Understanding:** This means your frontend (origin `.0.1:3000`) was trying to access your backend (origin `.0.1:8001`), but the backend's server didn't explicitly allow requests from `.0.1:3000`.
 - **Solution:**
 1. **Frontend API URL Update:** You had to update `src/services/api/axiosInstance.ts` to point to the correct backend port: `baseURL: ".0.1:8001/api/v1"`. This was a crucial step before addressing CORS.
 2. **Backend CORS Configuration (Flask-CORS):**
 - Install `Flask-CORS` in your backend: `pip install flask-cors`.
 - Modify your Flask `app.py` or `run.py` to enable CORS:
 ```python
 from flask import Flask
 from flask_cors import CORS

 app = Flask(__name__)
 CORS(app)

# Enables CORS for all routes and origins (for development)

# Or for specific origins:

# CORS(app, origins=[".0.1:3000","])
 ```
 - **Verification:** After these changes and restarting both servers, the frontend should now successfully make API calls to the backend without CORS errors.

# **Related Concepts:**

---
- [[CORS (Cross-Origin Resource Sharing)]]
- [[Port Management]]
- Debugging Strategies
- Environment Variables
- Multi-root Workspaces
- [[Flask (web framework)]]
- [[React - Vite]]
- [[React - Axios]]
- [[Python - Celery]]
- [[Docker - What is Docker and Containers]] (relevant if running backend via Docker)
- [[API Endpoints]]

# **Examples:**

---
#

#

# Example: Flask Backend CORS Configuration (`app.py` or `run.py`)

```python

# app.py or run.py in InnoBee-Backend
from flask import Flask
from flask_cors import CORS

# Import the CORS extension
import os

# To access environment variables

# Assuming your app initialization logic
app = Flask(__name__)

#
---
ADD CORS CONFIGURATION
---
# This line enables Cross-Origin Resource Sharing (CORS) for your Flask application.

# It allows requests from different origins (like your frontend running on )

# to access your backend API.

# During development, CORS(app) allows all origins, which is convenient but less secure for production.

# For more security, you can specify allowed origins:

# CORS(app, origins=[".0.1:3000", "])
CORS(app)

#
---

PORT CONFIGURATION
---
# This gets the port from the FLASK_PORT environment variable,

# defaulting to 8000 if the variable is not set.

# This makes your Flask app flexible to run on different ports (e.g., 8001 as we did).
port = int(os.environ.get('FLASK_PORT', 8000))

# Example route (ensure this exists in your app)
@app.route('/')
def home:
 return "Hello from InnoBee Backend!"

# Your existing main execution block
if __name__ == '__main__':

# Flask is running on .0.1:8001 (Debug: True)

# The port variable ensures it picks up FLASK_PORT from the environment.
 app.run(host='127.0.1', port=port, debug=True)
```

#

#

# Example: Frontend API Base URL Configuration (`src/services/api/axiosInstance.ts`)

```typescript
// src/services/api/axiosInstance.ts in InnoBee frontend
import axios from 'axios';

//
---
UPDATE BASE_URL
---
// This defines the base URL for all API requests made by this axios instance.
// It's crucial that this matches the address and port where your backend Flask API is running.
// We changed this from :8000 to :8001 during our troubleshooting.
const baseURL = ".0.1:8001/api/v1";

const axiosInstance = axios.create({
 baseURL: baseURL,
 headers: {
 'Content-Type': 'application/json',
 // You might add Authorization headers here later for authentication
 },
});

export default axiosInstance;
```

# **Flashcards:**

---
What is the primary reason for a "No 'Access-Control-Allow-Origin' header is present" error?;; It's a **CORS error**, meaning the backend server (e.g., Flask) isn't configured to allow requests from the frontend's origin (e.g., `).

How do you run a Flask backend on a specific port (like 8001) from the terminal in a way that respects environment variables?;; `FLASK_PORT=8001 python3 run.py` (assuming your Flask app reads `FLASK_PORT`).

What is the benefit of using a multi-root workspace (e.g., in Cursor or VS Code) for full-stack development?;; It allows you to easily navigate, search, and for AI tools to understand context across both your frontend and backend repositories simultaneously.