---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date:
last-reviewed: 2025-08-29
scheda: done
visit-count: 5
confidence-level: 2
consecutive-correct: 3
last-struggle-date: 2025-07-20

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
JSON (JavaScript Object Notation) solves the fundamental problem of ==data interchange between systems, applications, and programming languages in a human-readable, lightweight format==. It provides a standardized way to ==serialize JavaScript objects into strings for storage, transmission over networks, and communication with APIs==. JSON is crucial in modern web development because ==it's the de facto standard for REST APIs, configuration files, data storage, and client-server communication, offering a simple alternative to more verbose formats like [[React - JSX (JavaScript XML)]] while maintaining language independence and easy parsing.==

# **Core Explanation:**

---
#

# *JSON is a text-based data interchange format with specific syntax rules and built-in JavaScript methods for conversion:*

**JSON Syntax Rules:**
- **Data types**: Supports strings, numbers, booleans, null, objects, and arrays
- **Strings**: Must use double quotes, not single quotes
- **Objects**: Key-value pairs where keys must be strings in double quotes
- **Arrays**: Ordered lists of values enclosed in square brackets
- **No functions**: Cannot contain JavaScript functions, undefined, or comments

**JavaScript JSON Methods:**
>- **JSON.stringify**: Converts JavaScript objects/values to JSON strings
>- **JSON.parse**: Converts JSON strings back to JavaScript objects/values
>- **Error handling**: Both methods can throw SyntaxError for invalid input

**Key Differences from JavaScript Objects:**
- **Quoted keys**: Object keys must be <mark style="background:

# ADCCFFA6;">strings in double quotes</mark>
- **Limited types**: <mark style="background:

# BBFABBA6;">No undefined, functions, Symbol, or Date objects (serialized as strings)</mark>
- **No trailing commas**: Strict syntax requirements
- **No comments**: Pure data format <mark style="background:

# FF5582A6;">without documentation</mark>

*Key characteristics: Language-independent despite the name, lightweight and fast to parse, strict syntax requirements, widely supported across platforms, and designed specifically for data exchange rather than code execution.*

# **Related Concepts:**

---
#

# *JSON connects to several important programming and web development concepts:*

- **Serialization/Deserialization**: Converting objects to strings and back
- **REST APIs**: Primary data format for HTTP API communication
- **AJAX/Fetch**: Sending and receiving JSON data in web requests
- **Data Persistence**: Storing application state and configuration
- **NoSQL Databases**: Many use JSON-like document structures
- **Configuration Files**: [[package.json]], config files, settings
- **Web Storage**: localStorage and sessionStorage with JSON
- **Data Validation**: JSON Schema for validating JSON structure
- **Cross-platform Communication**: Language-agnostic data exchange

# **Examples:**

---
```javascript
// BASIC JSON STRUCTURE AND SYNTAX

console.log('=== JSON Structure Examples ===');

// Valid JSON examples (as strings)
const validJsonString = `{
 "name": "John Doe",
 "age": 30,
 "isActive": true,
 "address": {
 "street": "123 Main St",
 "city": "New York",
 "zipCode": "10001"
 },
 "hobbies": ["reading", "swimming", "coding"],
 "spouse": null
}`;

// Invalid JSON examples (would cause errors)
const invalidJsonExamples = [
 `{ name: "John" }`, // Keys must be quoted
 `{ "name": 'John' }`, // Values must use double quotes
 `{ "age": undefined }`, // undefined not allowed
 `{ "func": function {} }`, // functions not allowed
 `{ "trailing": "comma", }`, // trailing commas not allowed
];

console.log('Valid JSON structure demonstrated above');

// JSON.STRINGIFY - Converting JavaScript to JSON

console.log('=== JSON.stringify Examples ===');

// Basic object stringification
const user = {
 id: 1,
 name: 'Alice Johnson',
 email: 'alice@example.com',
 isAdmin: false,
 loginCount: 42,
 lastLogin: new Date('2025-06-30'),
 preferences: {
 theme: 'dark',
 notifications: true
 },
 tags: ['developer', 'javascript', 'frontend']
};

const userJson = JSON.stringify(user);
console.log('Stringified user:', userJson);
// Note: Date becomes ISO string, functions would be omitted

// Stringify with formatting (pretty printing)
const prettyJson = JSON.stringify(user, null, 2); // 2 spaces indentation
console.log('Pretty formatted JSON:');
console.log(prettyJson);

// Stringify specific properties only (replacer array)
const publicUserJson = JSON.stringify(user, ['id', 'name', 'email']);
console.log('Public user data:', publicUserJson);
// Only includes specified properties

// Stringify with custom replacer function
const customJson = JSON.stringify(user, (key, value) => {
 // Custom logic for each key-value pair
 if (key === 'email') {
 return value.toLowerCase; // Ensure email is lowercase
 }
 if (key === 'lastLogin') {
 return value.toISOString; // Custom date formatting
 }
 if (typeof value === 'string') {
 return value.trim; // Trim all string values
 }
 return value; // Return unchanged for other types
});
console.log('Custom processed JSON:', customJson);

// Handling special values
const specialValues = {
 regularString: 'hello',
 emptyString: '',
 zeroNumber: 0,
 nullValue: null,
 undefinedValue: undefined, // Will be omitted
 functionValue: function {}, // Will be omitted
 dateValue: new Date,
 arrayWithUndef: [1, undefined, 3] // undefined becomes null in arrays
};

console.log('Special values stringified:');
console.log(JSON.stringify(specialValues, null, 2));

// JSON.PARSE - Converting JSON strings to JavaScript

console.log('=== JSON.parse Examples ===');

// Basic parsing
const jsonString = '{"name": "Bob", "age": 25, "skills": ["JavaScript", "Python"]}';
const parsedObject = JSON.parse(jsonString);
console.log('Parsed object:', parsedObject);
console.log('Name:', parsedObject.name, 'Type:', typeof parsedObject);

// Parsing arrays
const jsonArray = '[{"id": 1, "name": "Item 1"}, {"id": 2, "name": "Item 2"}]';
const parsedArray = JSON.parse(jsonArray);
console.log('Parsed array:', parsedArray);
console.log('First item:', parsedArray);

// Parse with reviver function (custom processing)
const jsonWithDates = '{"created": "2025-06-30T10:00:00Z", "updated": "2025-06-30T15:30:00Z", "count": 5}';
const objectWithDates = JSON.parse(jsonWithDates, (key, value) => {
 // Convert ISO date strings back to Date objects
 if (key.includes('created') || key.includes('updated')) {
 return new Date(value);
 }
 return value;
});
console.log('Object with converted dates:', objectWithDates);
console.log('Created is Date object:', objectWithDates.created instanceof Date);

// ERROR HANDLING

console.log('=== Error Handling ===');

// Handling JSON.parse errors
function safeJsonParse(jsonString) {
 try {
 return { success: true, data: JSON.parse(jsonString) };
 } catch (error) {
 return { success: false, error: error.message };
 }
}

// Valid JSON
const validResult = safeJsonParse('{"valid": "json"}');
console.log('Valid parse result:', validResult);

// Invalid JSON
const invalidResult = safeJsonParse('{"invalid": json}'); // Missing quotes
console.log('Invalid parse result:', invalidResult);

// Handling JSON.stringify errors (circular references)
const circularObject = { name: 'test' };
circularObject.self = circularObject; // Create circular reference

function safeJsonStringify(obj) {
 try {
 return { success: true, json: JSON.stringify(obj) };
 } catch (error) {
 return { success: false, error: error.message };
 }
}

const circularResult = safeJsonStringify(circularObject);
console.log('Circular reference error:', circularResult);

// WORKING WITH APIs AND HTTP REQUESTS

console.log('=== API Integration Examples ===');

// Simulated API request/response handling
function simulateApiCall(endpoint, data) {
 console.log(`Making API call to: ${endpoint}`);

 // Convert JavaScript object to JSON for sending
 const requestBody = JSON.stringify(data);
 console.log('Request body (JSON):', requestBody);

 // Simulate server response (JSON string)
 const serverResponse = `{
 "success": true,
 "data": {
 "id": 123,
 "message": "Data processed successfully",
 "timestamp": "${new Date.toISOString}"
 }
 }`;

 // Parse JSON response back to JavaScript object
 const responseData = JSON.parse(serverResponse);
 console.log('Parsed response:', responseData);

 return responseData;
}

// Example API usage
const apiData = {
 username: 'john_doe',
 email: 'john@example.com',
 action: 'create_account'
};

const result = simulateApiCall('/api/users', apiData);

// Modern fetch API example (commented as it requires actual HTTP)
/*
async function fetchUserData(userId) {
 try {
 const response = await fetch(`/api/users/${userId}`);
 const userData = await response.json; // Automatically parses JSON
 return userData;
 } catch (error) {
 console.error('Failed to fetch user data:', error);
 }
}
*/

// LOCAL STORAGE WITH JSON

console.log('=== Local Storage Integration ===');

// Local storage only stores strings, so we use JSON for objects
function saveToStorage(key, data) {
 try {
 const jsonString = JSON.stringify(data);
 // localStorage.setItem(key, jsonString); // Would work in browser
 console.log(`Would save to localStorage["${key}"]:`);
 console.log(jsonString);
 return true;
 } catch (error) {
 console.error('Failed to save to storage:', error);
 return false;
 }
}

function loadFromStorage(key) {
 try {
 // const jsonString = localStorage.getItem(key); // Would work in browser
 const jsonString = '{"theme": "dark", "language": "en", "lastVisit": "2025-06-30"}';

 if (jsonString === null) return null;
 return JSON.parse(jsonString);
 } catch (error) {
 console.error('Failed to load from storage:', error);
 return null;
 }
}

// Example usage
const userSettings = {
 theme: 'dark',
 language: 'en',
 notifications: true,
 lastVisit: new Date
};

saveToStorage('userSettings', userSettings);
const loadedSettings = loadFromStorage('userSettings');
console.log('Loaded settings:', loadedSettings);

// DEEP CLONING WITH JSON

console.log('=== Deep Cloning ===');

// JSON can be used for deep cloning (with limitations)
const originalObject = {
 name: 'Original',
 nested: {
 value: 42,
 array: [1, 2, 3]
 }
};

// Deep clone using JSON (loses functions, undefined, etc.)
function deepCloneJson(obj) {
 return JSON.parse(JSON.stringify(obj));
}

const clonedObject = deepCloneJson(originalObject);
clonedObject.name = 'Cloned';
clonedObject.nested.value = 100;

console.log('Original:', originalObject);
console.log('Cloned:', clonedObject);
console.log('Objects are independent:', originalObject.nested.value === 42);

// CONFIGURATION FILES EXAMPLE

console.log('=== Configuration Management ===');

// Typical package.json structure
const packageConfig = {
 name: 'my-awesome-app',
 version: '1.0',
 description: 'An awesome JavaScript application',
 main: 'index.js',
 scripts: {
 start: 'node index.js',
 test: 'jest',
 build: 'webpack --mode production'
 },
 dependencies: {
 express: '^4.0',
 lodash: '^4.21'
 },
 devDependencies: {
 jest: '^29.0',
 webpack: '^5.0'
 },
 keywords: ['javascript', 'node', 'web'],
 author: 'Your Name',
 license: 'MIT'
};

console.log('Package.json example:');
console.log(JSON.stringify(packageConfig, null, 2));

// Application configuration
const appConfig = {
 server: {
 port: process.env.PORT || 3000,
 host: 'localhost'
 },
 database: {
 url: 'mongodb://localhost:27017/myapp',
 options: {
 useUnifiedTopology: true
 }
 },
 features: {
 authentication: true,
 logging: true,
 caching: false
 }
};

// Load configuration function
function loadConfiguration(configJson) {
 const config = JSON.parse(configJson);

 // Validate required fields
 if (!config.server || !config.server.port) {
 throw new Error('Server port is required in configuration');
 }

 return config;
}

// DATA VALIDATION

console.log('=== JSON Data Validation ===');

// Simple JSON schema validation function
function validateUserJson(jsonString) {
 try {
 const user = JSON.parse(jsonString);

 // Required fields validation
 const requiredFields = ['name', 'email', 'age'];
 for (const field of requiredFields) {
 if (!(field in user)) {
 return { valid: false, error: `Missing required field: ${field}` };
 }
 }

 // Type validation
 if (typeof user.name !== 'string' || user.name.trim === '') {
 return { valid: false, error: 'Name must be a non-empty string' };
 }

 if (typeof user.age !== 'number' || user.age < 0 || user.age > 150) {
 return { valid: false, error: 'Age must be a number between 0 and 150' };
 }

 // Email format validation (simple)
 const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
 if (!emailRegex.test(user.email)) {
 return { valid: false, error: 'Invalid email format' };
 }

 return { valid: true, data: user };

 } catch (error) {
 return { valid: false, error: `Invalid JSON: ${error.message}` };
 }
}

// Test validation
const validUserJson = '{"name": "Alice", "email": "alice@example.com", "age": 25}';
const invalidUserJson = '{"name": "", "email": "invalid-email", "age": -5}';

console.log('Valid user validation:', validateUserJson(validUserJson));
console.log('Invalid user validation:', validateUserJson(invalidUserJson));

// PERFORMANCE CONSIDERATIONS

console.log('=== Performance Tips ===');

// Large object stringification timing
const largeObject = {
 users: Array.from({ length: 1000 }, (_, i) => ({
 id: i,
 name: `User ${i}`,
 email: `user${i}@example.com`,
 created: new Date
 }))
};

console.time('Large object stringify');
const largeJson = JSON.stringify(largeObject);
console.timeEnd('Large object stringify');

console.time('Large JSON parse');
const parsedLarge = JSON.parse(largeJson);
console.timeEnd('Large JSON parse');

console.log(`JSON string size: ${largeJson.length} characters`);
console.log(`Parsed object has ${parsedLarge.users.length} users`);
````

# **Flashcards:**

---
What are the main differences between JSON and JavaScript objects?;; JSON requires double quotes for keys, doesn't support functions/undefined/comments, has stricter syntax (no trailing commas), and is a text format for data exchange

What does JSON.stringify do and what are its three parameters?;; Converts JavaScript values to JSON strings. Parameters: (value, replacer, space) - replacer can filter properties, space adds formatting/indentation

What does JSON.parse do and how do you handle parsing errors?;; Converts JSON strings to JavaScript values. Handle errors with try-catch blocks since it throws SyntaxError for invalid JSON

What JavaScript data types are NOT supported in JSON?;; Functions, undefined, Symbol, comments, and trailing commas are not supported. Date objects become ISO strings

How do you create formatted (pretty-printed) JSON?;; Use JSON.stringify(obj, null, 2) - the third parameter adds indentation (2 spaces in this example)

What happens to undefined values in JSON.stringify?;; In objects: undefined properties are omitted. In arrays: undefined becomes null. This maintains array indices but removes object properties