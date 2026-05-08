---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date: ""
last-reviewed: 2025-07-20
scheda: done
visit-count: 2
confidence-level: 2
consecutive-correct: 2

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

# Purpose/Why:

---
JavaScript's interaction functions (alert, prompt, confirm) solve the fundamental problem of ==basic user communication in web applications by providing simple, built-in methods for displaying information and collecting user input without requiring complex HTML or CSS==.

These functions address the need for immediate, synchronous user interaction that blocks execution until the user responds, making them essential for <mark style="background:

# FF5582A6;">simple data validation</mark>, user confirmation, debugging, and basic user experience flows. They are important because they provide a universal, cross-browser way to create modal dialogs that work consistently across different platforms and devices, offering developers a <mark style="background:

# FF5582A6;">quick solution for basic user interaction</mark> without the overhead of custom modal implementations.

While limited in customization, they serve as the foundation for understanding user interaction patterns and are particularly valuable for prototyping, learning, and situations where simplicity is preferred over advanced UI design.

# Core Explanation:

---

JavaScript's interaction functions are browser-provided methods that create modal windows for communicating with users. A modal window is a dialog that blocks interaction with the rest of the page until dismissed, ensuring the user must respond before continuing.

**Alert** displays a message with a single "OK" button, primarily used for notifications or debugging. It accepts one parameter (the message) and returns `undefined`, pausing script execution until acknowledged.

**Prompt** shows a message with an input field and OK/Cancel buttons, used for collecting text input from users. It accepts two parameters: a required title/message and an optional default value. It returns the entered text as a string, or `null` if cancelled via Cancel button or Escape key.

**Confirm** presents a yes/no question with OK/Cancel buttons, used for user confirmations before actions like deletions or navigation. It accepts one parameter (the question) and returns a boolean value: `true` for OK, `false` for Cancel/Escape.

All three functions are synchronous, blocking JavaScript execution until the user responds. They have browser-dependent styling and positioning that cannot be customized through JavaScript or CSS. The appearance and exact behavior may vary between browsers, with some legacy browsers (like Internet Explorer) having specific quirks that require workarounds.

# Related Concepts:

---
**[[JavaScript Modal Windows and User Experience]]**: These functions demonstrate the concept of modal interactions in web development, where user attention is focused on a specific task before proceeding, forming the basis for more advanced modal dialog patterns.

**Synchronous vs Asynchronous JavaScript**: Unlike most modern JavaScript APIs, these functions are synchronous, blocking code execution until user interaction completes, contrasting with promise-based and callback-driven asynchronous patterns.

**Browser APIs and Web Standards**: These functions are part of the browser's host environment rather than core JavaScript, representing how browsers extend the language with platform-specific functionality for web interaction.

**Event-Driven Programming**: While these functions block execution, they relate to event-driven programming concepts where user actions trigger responses, though they use an older synchronous model rather than modern event listeners.

**Form Validation and Input Handling**: Prompt and confirm functions provide basic input collection and validation, serving as predecessors to more sophisticated form handling techniques using HTML5 validation and custom JavaScript validation.

**DOM Manipulation Alternatives**: These functions offer an alternative to creating custom HTML elements and manipulating the DOM for simple user interactions, though with limited flexibility compared to modern approaches.

# Examples:

---
```javascript
// Example 1: Basic alert usage for notifications and debugging
function demonstrateAlert {
 // Simple message display - blocks execution until user clicks OK
 alert("Welcome to our website!"); // User must click OK to continue

 // Debugging usage - showing variable values during development
 let debugValue = "Current state: processing";
 alert(`Debug info: ${debugValue}`); // Template literal for dynamic messages

 // Alert with longer messages and special characters
 alert("Operation completed successfully!\nClick OK to continue."); // \n for line breaks

 console.log("This line executes only after alert is dismissed");
}

// Example 2: Prompt for user input collection
function demonstratePrompt {
 // Basic prompt with question only
 let userName = prompt("What is your name?"); // Returns string or null

 // Check if user cancelled (returned null)
 if (userName === null) {
 alert("You cancelled the input!"); // Handle cancellation
 return; // Exit function early
 } else if (userName === "") {
 alert("You entered an empty name!"); // Handle empty input
 } else {
 alert(`Hello, ${userName}!`); // Use the collected input
 }

 // Prompt with default value - second parameter provides initial text
 let userAge = prompt("How old are you?", "25"); // "25" appears in input field

 // Convert string result to number for calculations
 if (userAge !== null) { // User didn't cancel
 let age = parseInt(userAge); // Convert string to integer
 if (isNaN(age)) { // Check if conversion failed
 alert("Please enter a valid number next time!");
 } else {
 alert(`In 10 years, you'll be ${age + 10} years old!`);
 }
 }

 // Internet Explorer compatibility - always provide default to avoid "undefined"
 let feedback = prompt("Any feedback?", ""); // Empty string prevents "undefined" in IE
}

// Example 3: Confirm for user decisions and validation
function demonstrateConfirm {
 // Basic yes/no confirmation
 let shouldContinue = confirm("Do you want to continue?"); // Returns boolean

 if (shouldContinue) { // true if OK pressed
 alert("You chose to continue!");
 // Continue with the operation
 } else { // false if Cancel pressed or Escape key
 alert("Operation cancelled by user");
 return; // Exit function or redirect user
 }

 // Confirmation before destructive actions
 let confirmDelete = confirm("Are you sure you want to delete this item? This cannot be undone.");
 if (confirmDelete) {
 // Perform deletion logic here
 alert("Item deleted successfully!");
 } else {
 alert("Deletion cancelled - item preserved");
 }

 // Chaining confirmations for critical operations
 if (confirm("Delete all data?")) {
 if (confirm("This will permanently delete everything. Are you absolutely sure?")) {
 alert("All data deleted!"); // Only executes after two confirmations
 } else {
 alert("Final confirmation cancelled");
 }
 }
}

// Example 4: Practical application - Simple calculator with user interaction
function interactiveCalculator {
 // Get operation type from user
 let operation = prompt("Choose operation: +, -, *, /", "+");

 if (operation === null) return; // User cancelled

 // Validate operation
 if (!["+", "-", "*", "/"].includes(operation)) {
 alert("Invalid operation! Please use +, -, *, or /");
 return;
 }

 // Get first number
 let num1 = prompt("Enter first number:", "0");
 if (num1 === null) return; // User cancelled
 num1 = parseFloat(num1); // Convert to number

 if (isNaN(num1)) {
 alert("First number is invalid!");
 return;
 }

 // Get second number
 let num2 = prompt("Enter second number:", "0");
 if (num2 === null) return; // User cancelled
 num2 = parseFloat(num2); // Convert to number

 if (isNaN(num2)) {
 alert("Second number is invalid!");
 return;
 }

 // Check for division by zero
 if (operation === "/" && num2 === 0) {
 alert("Error: Division by zero is not allowed!");
 return;
 }

 // Perform calculation
 let result;
 switch (operation) {
 case "+": result = num1 + num2; break;
 case "-": result = num1 - num2; break;
 case "*": result = num1 * num2; break;
 case "/": result = num1 / num2; break;
 }

 // Show result and ask if user wants to continue
 alert(`Result: ${num1} ${operation} ${num2} = ${result}`);

 if (confirm("Do you want to perform another calculation?")) {
 interactiveCalculator; // Recursive call for another calculation
 } else {
 alert("Calculator session ended. Thank you!");
 }
}

// Example 5: Form validation and error handling patterns
function validateUserRegistration {
 // Collect required information with validation
 let email = prompt("Enter your email address:");
 if (!email || email === null) {
 alert("Email is required for registration!");
 return false;
 }

 // Basic email validation using simple pattern check
 if (!email.includes("@") || !email.includes(".")) {
 alert("Please enter a valid email address!");
 return false;
 }

 // Password with minimum length requirement
 let password = prompt("Create a password (minimum 6 characters):");
 if (!password || password === null) {
 alert("Password is required!");
 return false;
 }

 if (password.length < 6) {
 alert("Password must be at least 6 characters long!");
 return false;
 }

 // Confirmation step
 if (confirm(`Register with email: ${email}?`)) {
 alert("Registration successful! Welcome aboard!");
 return true;
 } else {
 alert("Registration cancelled");
 return false;
 }
}

// Example usage of the functions
// demonstrateAlert;
// demonstratePrompt;
// demonstrateConfirm;
// interactiveCalculator;
// validateUserRegistration;

// Note: These functions block execution, so uncomment one at a time for testing
console.log("All interaction functions are defined and ready to use");
```

# Flashcards:

---
What are the three JavaScript interaction functions and what does each return?;; alert shows a message and returns undefined, prompt collects input and returns a string or null, confirm asks yes/no and returns true or false

What happens when a user cancels a prompt or confirm dialog?;; prompt returns null when cancelled, confirm returns false when cancelled or Escape is pressed, both allow checking for user cancellation in code

What is a modal window and why do alert, prompt, and confirm create them?;; A modal window blocks interaction with the rest of the page until dismissed, ensuring users must respond to the dialog before continuing script execution, making these functions synchronous