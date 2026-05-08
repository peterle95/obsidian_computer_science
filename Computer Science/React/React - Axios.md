---
memory: to_finish
tags:
 - will_learn
language:
 - React
review-date: ""
last-reviewed: 2025-07-19
keywords:
 - HTTP
 - API
 - frontend
 - backend
 - CORS
 - baseURL
 - interceptors
 - JavaScript
 - TypeScript
scheda: done
visit-count: 3
confidence-level: 1
consecutive-correct: 0
last-struggle-date: 2025-07-19

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
Axios is a popular, ==React-based **promise-based HTTP client** for the browser and Node.js==. ==It simplifies making HTTP requests to external resources like APIs. While browsers have a built-in `fetch` API, Axios offers a more convenient and feature-rich experience, especially for larger applications. It handles various aspects of HTTP communication, including request/response interception, automatic JSON data transformation, and robust error handling.==

==Axios is not a REST API. Instead, it is an HTTP client library that can interact with any REST API by making HTTP requests, such as GET, POST, PUT, or DELETE==

>In the context of our InnoBee project, Axios is used in the frontend (React/Vite) to communicate with the Flask backend API.

--> further reading

# **Related Concepts:**

---
- **HTTP Methods:** Axios uses standard HTTP methods like `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, and `OPTIONS` to interact with RESTful APIs.
- **Promises:** Axios is built on JavaScript Promises, which are objects representing the eventual completion (or failure) of an asynchronous operation and its resulting value. This makes asynchronous code easier to write and manage.
- **CORS (Cross-Origin Resource Sharing):** A security mechanism implemented in web browsers that restricts web pages from making requests to a different domain than the one that served the web page. This is why we encountered the "blocked by CORS policy" error when the frontend on port 3000 tried to access the backend on port 8001 without proper backend configuration.
- **`baseURL`**: A core configuration option in Axios that allows you to define a base URL for all requests made using a specific Axios instance. This avoids repeating the full URL for every API call and simplifies API management.
- **API Endpoints**: Specific URLs on the backend that perform particular actions (e.g., `/api/v1/auth/login`). Axios sends requests to these endpoints.
- **Frontend-Backend Communication**: The process by which the client-side application (InnoBee React app) sends data to and receives data from the server-side application (InnoBee Flask API).

# **Examples:**

---
#

#

# Axios Setup and Fixing the `baseURL`

We initially had a CORS error because the frontend's Axios configuration was pointing to `.0.1:8000/api/v1`, while the backend was running on `.0.1:8001/api/v1`. The fix involved updating the `baseURL` in the frontend.
```TypeScript
// File: src/services/api/axiosInstance.ts (InnoBee Frontend)

import axios from 'axios';

// Create an instance of Axios with a base URL
// This is crucial for defining where your frontend will send API requests.
// Initially, this was set to port 8000, causing a connection issue when the
// backend was running on 8001.
const axiosInstance = axios.create({
 baseURL: '.0.1:8001/api/v1', // Corrected to point to the backend's actual port
 timeout: 10000, // Request timeout in milliseconds (e.g., 10 seconds)
 headers: {
 'Content-Type': 'application/json', // Default content type for requests
 },
});

// Request Interceptor: Optional, but useful for adding headers (like auth tokens)
// before each request is sent.
axiosInstance.interceptors.request.use(
 (config) => {
 // Example: Add an Authorization token from localStorage if available
 const token = localStorage.getItem('authToken');
 if (token) {
 config.headers.Authorization = `Bearer ${token}`;
 }
 return config;
 },
 (error) => {
 // Handle request error
 return Promise.reject(error);
 }
);

// Response Interceptor: Optional, for handling responses globally (e.g., refreshing tokens,
// logging errors, or redirecting on certain status codes).
axiosInstance.interceptors.response.use(
 (response) => {
 // Any status code that lies in the range of 2xx cause this function to trigger.
 return response;
 },
 (error) => {
 // Any status codes that fall outside the range of 2xx cause this function to trigger.
 if (error.response) {
 // The request was made and the server responded with a status code
 // that falls out of the range of 2xx
 console.error("API Error Response:", error.response.data);
 console.error("Status:", error.response.status);
 console.error("Headers:", error.response.headers);
 if (error.response.status === 401) {
 // Handle unauthorized errors, e.g., redirect to login
 // window.location.href = '/login';
 }
 } else if (error.request) {
 // The request was made but no response was received
 console.error("API Error Request:", error.request);
 } else {
 // Something happened in setting up the request that triggered an Error
 console.error("Error Message:", error.message);
 }
 return Promise.reject(error);
 }
);

export default axiosInstance;
```

#

#

# Example API Usage in a React Component

```TypeScript
// File: src/pages/MySubmission/MySubmission.tsx (Example usage)

import React, { useEffect, useState } from 'react';
import axiosInstance from '../../services/api/axiosInstance'; // Import our configured Axios instance

interface User {
 id: string;
 name: string;
 email: string;
}

const MySubmission: React.FC = => {
 const [users, setUsers] = useState<User>();
 const [loading, setLoading] = useState<boolean>(true);
 const [error, setError] = useState<string | null>(null);

 useEffect( => {
 const fetchUsers = async => {
 try {
 // Use the axiosInstance to make a GET request to the /users endpoint.
 // The baseURL ('.0.1:8001/api/v1') is automatically prepended.
 const response = await axiosInstance.get('/users');
 setUsers(response.data); // Assuming the API returns an array of users in the response.data
 } catch (err: any) {
 // Handle errors during the API call.
 // This leverages the error interceptor defined in axiosInstance.ts.
 setError(err.message || 'Failed to fetch users.');
 } finally {
 setLoading(false); // End loading regardless of success or failure.
 }
 };

 fetchUsers; // Call the async function to fetch data when the component mounts.
 }, ); // Empty dependency array means this effect runs once after the initial render.

 if (loading) {
 return <div className="p-4 text-center">Loading users...</div>;
 }

 if (error) {
 return <div className="p-4 text-center text-red-500">Error: {error}</div>;
 }

 return (
 <div className="p-4">
 <h1 className="text-2xl font-bold mb-4">My Submissions (Users)</h1>
 {users.length > 0 ? (
 <ul className="list-disc pl-5">
 {users.map((user) => (
 <li key={user.id} className="mb-2">
 <strong>{user.name}</strong> ({user.email})
 </li>
 ))}
 </ul>
 ) : (
 <p>No users found.</p>
 )}
 </div>
 );
};

export default MySubmission;
```

# **Flashcards:**

---
What is Axios and why is it used instead of `fetch`?;; Axios is a promise-based HTTP client for making API requests. It's often preferred over `fetch` for its richer features like interceptors, automatic JSON transformation, and better error handling.

What caused the "blocked by CORS policy" error with Axios in our InnoBee project?;; The frontend (on port 3000) was trying to make a request to the backend (on port 8001) without the backend being configured to allow cross-origin requests from the frontend's origin.

How do you fix a `baseURL` configuration issue in Axios like we did?;; Locate the `axios.create` instance (e.g., `axiosInstance.ts`) and modify the `baseURL` property to point to the correct backend address and port (e.g., `.0.1:8001/api/v1`).