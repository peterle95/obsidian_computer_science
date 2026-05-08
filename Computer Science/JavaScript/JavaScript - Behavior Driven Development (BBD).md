---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date:
last-reviewed: 2025-08-24
scheda: done
visit-count: 3
confidence-level: 1
consecutive-correct: 1
last-struggle-date: 2025-08-14

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
Behavior Driven Development (BDD) and testing frameworks like Mocha solve the problem of code reliability and maintainability. They provide a ==structured approach to ensure code works as expected== and continues to work after changes. This is crucial in software development because:

1. It prevents bugs from reaching users
2. It enables safer code modifications and refactoring
3. It enforces better code architecture from the beginning
4. It serves as living documentation for how code should behave

# Core Explanation:

---

In BDD, the specification (spec) goes first, followed by implementation. The testing structure uses two primary functions:

>- `describe`: Creates a group of tests, organizing them into logical sections or units
>- `it`: Contains an actual test case representing a specific behavior the code should exhibit

This structure serves three purposes:
>1. As Tests - they guarantee that code works correctly
>2. As Documentation - the titles of describe and it blocks explain what the function does
>3. As Examples - tests demonstrate how to use the code

With proper tests, developers can safely improve, change, or even rewrite functions while ensuring they still work correctly. This is especially important in large projects where manually checking every place a function is used becomes impossible.

Without tests, developers either:
1. Make changes and risk introducing bugs
2. Avoid modifying code out of fear, leading to outdated and unmaintained code

Automatic testing helps avoid these problems by providing quick verification of all functionality after changes.

# Related Concepts:

---
- **Test-Driven Development (TDD)**: Similar to BDD but focuses more on testing technical implementation rather than behavior. TDD emphasizes writing tests before code but is less focused on readable specifications.

- **Mocha**: [[JavaScript Testing Frameworks (e.g., Jest, Mocha, Jasmine) - Overview]] A JavaScript testing framework that provides the `describe` and `it` functions for structuring tests. It's commonly used for BDD-style testing.

- **Assertions**: Statements that verify expected outcomes in tests. Libraries like Chai are often used with Mocha to provide assertion functionality.

- **Continuous Integration (CI)**: Process of automatically running tests when code changes are submitted, ensuring tests pass before code is merged.

- **Code Coverage**: Measurement of how much code is executed during tests, helping identify untested parts of the codebase.

# Examples:

---
```javascript
// Example of testing a power function with BDD style using Mocha

// 'describe' creates a test group for our power function
describe("pow", function {

 // Each 'it' block represents a specific behavior we want to test
 it("raises 2 to the power of 3", function {
 // The actual test assertion - we expect pow(2, 3) to equal 8
 assert.equal(pow(2, 3), 8);
 });

 // We can have multiple test cases to verify different behaviors
 it("raises 3 to the power of 4", function {
 assert.equal(pow(3, 4), 81);
 });

 // We can nest 'describe' blocks to organize related tests
 describe("negative exponents", function {
 it("raises 2 to the power of -1", function {
 assert.equal(pow(2, -1), 0.5);
 });
 });

 // We can also test edge cases
 it("returns 1 for any number raised to power 0", function {
 assert.equal(pow(5, 0), 1);
 assert.equal(pow(10, 0), 1);
 assert.equal(pow(0, 0), 1);
 });
});

// The actual implementation of the power function
function pow(base, exponent) {
 // Implementation goes here
 if (exponent < 0) {
 return 1 / Math.pow(base, -exponent);
 }
 return Math.pow(base, exponent);
}
```

What’s wrong in the test of `pow` below?

```javascript
it("Raises x to the power n", function {
 let x = 5;

 let result = x;
 assert.equal(pow(x, 1), result);

 result *= x;
 assert.equal(pow(x, 2), result);

 result *= x;
 assert.equal(pow(x, 3), result);
});
```

P.S. Syntactically the test is correct and passes.

The test demonstrates one of the temptations a developer meets when writing tests.

What we have here is actually 3 tests, but layed out as a single function with 3 asserts.

Sometimes it’s easier to write this way, but if an error occurs, it’s much less obvious what went wrong.

If an error happens in the middle of a complex execution flow, then we’ll have to figure out the data at that point. We’ll actually have to _debug the test_.

It would be much better to break the test into multiple `it` blocks with clearly written inputs and outputs.

Like this:

```javascript
describe("Raises x to power n", function {
 it("5 in the power of 1 equals 5", function {
 assert.equal(pow(5, 1), 5);
 });

 it("5 in the power of 2 equals 25", function {
 assert.equal(pow(5, 2), 25);
 });

 it("5 in the power of 3 equals 125", function {
 assert.equal(pow(5, 3), 125);
 });
});
```

We replaced the single `it` with `describe` and a group of `it` blocks. Now if something fails we would see clearly what the data was.

Also we can isolate a single test and run it in standalone mode by writing `it.only` instead of `it`:

```javascript
describe("Raises x to power n", function {
 it("5 in the power of 1 equals 5", function {
 assert.equal(pow(5, 1), 5);
 });

 // Mocha will run only this block
 it.only("5 in the power of 2 equals 25", function {
 assert.equal(pow(5, 2), 25);
 });

 it("5 in the power of 3 equals 125", function {
 assert.equal(pow(5, 3), 125);
 });
});
```

# Flashcards:

---
What is the purpose of the `describe` function in BDD testing?;; It creates a group of tests and organizes them into logical sections or units.

What is the purpose of the `it` function in BDD testing?;; It contains an actual test case that represents a specific behavior the code should exhibit.

What are the three ways a test spec can be used in BDD?;; 1. As Tests - to verify code works correctly, 2. As Documentation - to explain what functions do, 3. As Examples - to demonstrate how to use the code.

Why is writing tests before implementation beneficial?;; It ensures code is designed with testability in mind, creates clearer requirements, and leads to better architecture from the beginning.

What problem does automatic testing help solve in large projects?;; It prevents the dilemma between making changes that might introduce bugs or avoiding changes that lead to outdated code.

What is a key architectural benefit of writing tests?;; Tests require code to have clearly described tasks with well-defined inputs and outputs, leading to better overall architecture.