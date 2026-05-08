---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date:
last-reviewed: 2025-09-03
scheda: done
visit-count: 4
confidence-level: 2
consecutive-correct: 3
last-struggle-date: 2025-07-25

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
The nullish coalescing operator `??` solves the common problem of ==**providing a default value for a variable or expression only when that variable/expression is `null` or `undefined`**==. Historically, developers often used the `||` (OR) operator for this purpose. ==However, `||` considers `0`, `false`, and `""` (empty string) as "falsy" values, causing them to be replaced by the default, even when they might be legitimate and intended values. `??` specifically addresses this by distinguishing between these "falsy" but valid values and truly "undefined" or "null" (unknown/unset) values.== This precision is important for writing more robust and predictable code, especially when dealing with numerical inputs, boolean flags, or empty strings where `0`, `false`, or `""` are meaningful.

# **Core Explanation:**

---
The nullish coalescing operator, denoted by `??`, is a relatively recent addition to JavaScript (ES2020). It provides a concise way to select a default value.

The syntax is `a ?? b`. The result of this operation is:
* `a`, if `a` is **defined** (i.e., `a` is neither `null` nor `undefined`).
* `b`, if `a` is `null` or `undefined`.

==In simpler terms, `??` returns its first operand if it's not `null` or `undefined`; otherwise, it returns its second operand.==

**Key characteristics:**
* **Default Value Assignment**: Its most common use case is to provide a default value when a variable or expression might be `null` or `undefined`.
* **Distinction from `||`**: The crucial difference from the logical OR operator (`||`) is that `??` only treats `null` and `undefined` as values that trigger the fallback to the second operand. It **does not** treat `0`, `false`, or an empty string `""` as triggers for the default value, unlike `||`.
>* **Chaining**: Multiple `??` operators can be chained to select the first defined value from a list of variables or expressions: `value1 ?? value2 ?? value3 ?? defaultValue`.
* **Low Precedence**: The `??` operator has a low precedence (same as `||`), meaning it evaluates after most other operations like `+`, `-`, `*`, `/`. Parentheses are often required to ensure correct evaluation order in complex expressions.
* **Safety Restriction**: For safety, `??` cannot be directly combined with `&&` or `||` operators in the same expression without explicit parentheses. This avoids common logical errors when mixing these operators.

# **Related Concepts:**

---
* **Logical OR Operator (`||`)**: This is the most directly related operator. Historically, `||` was used to provide default values (e.g., `value = userSetting || defaultValue`). However, `||` evaluates to the first *truthy* value, meaning `0`, `false`, and `""` would also trigger the default. `??` was introduced to address this specific limitation, returning the first *defined* value instead.
* **`null` and `undefined`**: These two primitive values are central to how `??` operates. The operator specifically checks for these two values to determine if a fallback is needed.
* **Falsy Values**: A concept in JavaScript referring to values that evaluate to `false` in a boolean context (e.g., `false`, `0`, `""`, `null`, `undefined`, `NaN`). The distinction between `??` and `||` lies in how they handle falsy values: `||` triggers a fallback for all falsy values, while `??` only triggers for `null` and `undefined`.
* **Operator Precedence**: This concept dictates the order in which operators are evaluated in an expression. Understanding `??`'s low precedence is crucial, as it often requires parentheses when combined with arithmetic or other higher-precedence operators to ensure the desired evaluation order.
* **Ternary Operator (`? :`)**: While not identical, the `??` operator can often replace a more verbose ternary operation that checks for `null` and `undefined`: `(a !== null && a !== undefined) ? a : b` can be simplified to `a ?? b`.

# **Examples:**

---
```javascript
//
---
Basic Usage: Providing Default Values
---
let userStatus; // userStatus is undefined
console.log(userStatus ?? "Guest"); // "Guest" - userStatus is undefined, so "Guest" is returned.

let userName = "Alice";
console.log(userName ?? "Guest"); // "Alice" - userName is defined, so "Alice" is returned.

let userAge = null;
console.log(userAge ?? 18); // 18 - userAge is null, so 18 is returned.

//
---

Chaining ?? for multiple fallbacks
---
let firstName = null;
let lastName = null;
let nickName = "Supercoder";
let companyName = "Dev Solutions";

// Finds the first defined value in the sequence
console.log(firstName ?? lastName ?? nickName ?? companyName ?? "Anonymous User"); // "Supercoder"
// - firstName is null, moves to lastName.
// - lastName is null, moves to nickName.
// - nickName is "Supercoder" (defined), so it's returned.

//
---

Comparison with || (OR) operator
---
let height = 0; // 0 is a valid height, but it's a falsy value

// Using ||:
console.log("Using ||:");
console.log(height || 100); // 100 - `height` (0) is falsy, so 100 is returned. This might be undesirable.

// Using ?? :
console.log("Using ?? :");
console.log(height ?? 100); // 0 - `height` (0) is defined (not null/undefined), so 0 is returned. This is often the desired behavior.

let isActive = false; // false is a valid boolean state, but it's a falsy value

console.log("Using || with boolean:");
console.log(isActive || true); // true - `isActive` (false) is falsy, so true is returned.

console.log("Using ?? with boolean:");
console.log(isActive ?? true); // false - `isActive` (false) is defined, so false is returned.

let message = ""; // An empty string might be valid, but it's a falsy value

console.log("Using || with empty string:");
console.log(message || "Default Message"); // "Default Message" - `message` ("") is falsy, so "Default Message" is returned.

console.log("Using ?? with empty string:");
console.log(message ?? "Default Message"); // "" - `message` ("") is defined, so "" is returned.

//
---
Precedence with other operators
---
let settingWidth = null;
let settingHeight = null;

// Correct usage with parentheses due to low precedence of ??
let area = (settingWidth ?? 50) * (settingHeight ?? 20);
console.log(area); // 1000 - (50 * 20)

// Incorrect usage without parentheses - * has higher precedence
let badArea = settingWidth ?? 50 * settingHeight ?? 20;
// This evaluates as: settingWidth ?? (50 * settingHeight) ?? 20
// If settingWidth is null, then (50 * settingHeight) becomes (50 * null) which is 0.
// So, it effectively becomes null ?? 0 ?? 20, resulting in 0.
console.log(badArea); // 0 (demonstrates incorrect calculation)

//
---

Safety Restriction: Cannot mix ?? with && or || without parentheses
---
// let x = 1 && 2 ?? 3; // Uncommenting this line would cause a Syntax Error!
// console.log(x);

// Correct usage with explicit parentheses:
let y = (1 && 2) ?? 3; // (1 && 2) evaluates to 2. Then 2 ?? 3 evaluates to 2.
console.log(y); // 2

let z = 0 || (null ?? 5); // (null ?? 5) evaluates to 5. Then 0 || 5 evaluates to 5.
console.log(z); // 5
````

# **Flashcards:**

---
What is the purpose of the nullish coalescing operator ???;;To provide a default value for a variable or expression ONLY if it's null or undefined.

What is the key difference between ?? and ||?;;?? returns the first defined value (not null/undefined), while || returns the first truthy value (any non-falsy value).

Why might you need parentheses when using ?? with other operators like * or +?;;Because ?? has low precedence, similar to ||. Parentheses ensure the arithmetic/other operations are evaluated before the ?? operator.