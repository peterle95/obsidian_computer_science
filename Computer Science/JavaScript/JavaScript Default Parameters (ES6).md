---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date: ""
last-reviewed: 2025-07-21
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 3

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
JavaScript Default Parameters, introduced in ES6 (ECMAScript 2015), ==solve the problem of **handling missing or `undefined` arguments** passed to a function.== Before default parameters, developers had to manually check inside the function if an argument was provided and, if not, assign a default value using conditional statements (`if`) or the logical OR operator (`||`). This led to boilerplate code and potential issues if `0`, `false`, or an empty string `""` were valid inputs that `||` would incorrectly treat as missing. Default parameters provide a **cleaner, more concise, and robust syntax** for defining function parameters that will automatically take a specified default value if no argument is provided, or if the argument explicitly resolves to `undefined`. This improves code readability, reduces errors, and makes functions more flexible and easier to use.

# **Core Explanation:**

---
==Default parameters allow you to initialize function parameters with default values if no argument is provided or if the argument passed is `undefined`==. This feature streamlines the process of creating functions that can be called with varying numbers of arguments without needing extensive internal checks for missing values.

**How they work:**
1. **Syntax**: A default parameter is defined by assigning a default value directly in the parameter list during function declaration: `function funcName(param1 = defaultValue1, param2 = defaultValue2) { ... }`.
2. **Evaluation**: The default value expression is evaluated **at call time**, only if the corresponding argument is not provided or explicitly resolves to `undefined`.
3. **`undefined` vs. `null`**: It's crucial to understand that default parameters only take effect if the argument is missing or exactly `undefined`. If `null` is passed as an argument, the default value will *not* be used, as `null` is considered a defined value.
4. **Order**: Default parameters are typically placed at the end of the parameter list, making it clear which parameters are optional. While you can have non-default parameters after default ones, it often leads to less readable code as you'd need to pass `undefined` explicitly to skip a default parameter.
5. **Dynamic Defaults**: Default values can be the result of a function call or even reference other parameters defined earlier in the list.

Default parameters make function signatures more self-documenting and simplify the function body by removing the need for manual default value assignments.

# **Related Concepts:**

---
* **[[JavaScript Function Parameters and Arguments]]**: Default parameters are a direct extension of how function parameters and arguments work. They specifically enhance how arguments are handled when they are not provided or are `undefined`.
* **`undefined`**: This primitive value is central to default parameters. A default value is used *only* when the corresponding argument is `undefined` or completely omitted. This distinguishes it from `null`, `0`, `false`, or an empty string, which would *not* trigger the default value.
* **Logical OR Operator (`||`)**: Before ES6, `value = argument || defaultValue;` was a common idiom to set default values. However, `||` checks for *any falsy value* (`0`, `false`, `""`, `null`, `undefined`, `NaN`). Default parameters are more precise, only falling back for `null` or `undefined`, making them superior for scenarios where `0`, `false`, or `""` are valid inputs.
* **Function Overloading (or lack thereof in JS)**: While other languages allow defining multiple functions with the same name but different parameter lists (overloading), JavaScript handles optional parameters and varying argument counts through features like default parameters, rest parameters, and checking `arguments.length`. Default parameters offer a cleaner way to handle optional arguments compared to older methods.
* **[[JavaScript Functions]]**: Default parameters are a feature that enhances the way functions are defined and used, contributing to more robust and readable function signatures.

# **Examples:**

---
```javascript
//
---
1. Basic Default Parameter Usage
---
// Define a function 'greet' with a default value for 'name'
function greet(name = "Guest") {
 console.log(`Hello, ${name}!`);
}

// Call without providing the 'name' argument
greet; // Output: Hello, Guest! (Uses default value)

// Call with an argument
greet("Alice"); // Output: Hello, Alice! (Uses provided argument)

// Call with 'undefined' explicitly
greet(undefined); // Output: Hello, Guest! (Treats undefined as missing, uses default)

// Call with 'null' - IMPORTANT: null is NOT treated as missing
greet(null); // Output: Hello, null! (Null is a defined value, so the default is NOT used)

//
---
2. Multiple Default Parameters and Mixed Parameters
---
// Function to create a user profile with default values for 'age' and 'country'
function createUserProfile(username, age = 30, country = "Unknown") {
 console.log(`User: ${username}, Age: ${age}, Country: ${country}`);
}

// Provide only required argument
createUserProfile("Bob"); // Output: User: Bob, Age: 30, Country: Unknown

// Provide required argument and override first default
createUserProfile("Charlie", 25); // Output: User: Charlie, Age: 25, Country: Unknown

// Override all default arguments
createUserProfile("David", 40, "USA"); // Output: User: David, Age: 40, Country: USA

// To use a later default but skip an earlier one, pass 'undefined'
createUserProfile("Eve", undefined, "Canada"); // Output: User: Eve, Age: 30, Country: Canada
// - 'age' is undefined, so its default (30) is used.
// - 'country' is "Canada", so it overrides its default.

//
---
3. Dynamic Default Values (using previous parameters or function calls)
---
// Default value for 'message' depends on 'name'
function personalizeGreeting(name, message = `Welcome, ${name}!`) {
 console.log(message);
}

personalizeGreeting("Frank"); // Output: Welcome, Frank!
personalizeGreeting("Grace", "Good to see you, Grace!"); // Output: Good to see you, Grace!

// Default value from a function call
let getDefaultVolume = => 50;

function playAudio(track, volume = getDefaultVolume) {
 console.log(`Playing "${track}" at volume: ${volume}`);
}

playAudio("Song A"); // Output: Playing "Song A" at volume: 50
playAudio("Song B", 75); // Output: Playing "Song B" at volume: 75

//
---
4. Contrast with old || operator trick
---
let userHeight = 0; // A valid height

// Old way using ||
let displayHeightOld = userHeight || 100;
console.log(`Display Height (old ||): ${displayHeightOld}`); // Output: Display Height (old ||): 100
// Problem: 0 is falsy, so it gets replaced by 100, which might not be desired.

// New way using default parameters
function showHeight(h = 100) {
 console.log(`Display Height (default param): ${h}`);
}
showHeight(userHeight); // Output: Display Height (default param): 0
// Solution: 0 is a defined value, so it's kept as is. Default only applies for undefined/missing.
````

# **Flashcards:**

---
What is the purpose of default parameters in JavaScript (ES6)?;;To provide a fallback value for a function parameter if no argument is supplied or if the argument is undefined.

When does a default parameter's value get used?;;When the corresponding argument is either completely omitted during the function call or explicitly passed as undefined.

Does null trigger a default parameter to be used?;;No. null is considered a defined value, so passing null as an argument will use null itself, not the default.