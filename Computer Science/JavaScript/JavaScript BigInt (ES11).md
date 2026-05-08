---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date: ""
last-reviewed: 2025-07-21
scheda: done
visit-count: 4
confidence-level: 3
consecutive-correct: 4

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
`BigInt` is a primitive data type in JavaScript introduced in ECMAScript 2020 (ES11) that allows you to represent whole numbers larger than 253−1 (the maximum safe integer that the standard `number` type can accurately represent). Before `BigInt`, JavaScript's `number` type, being a 64-bit floating-point representation (IEEE 754), could only reliably handle integers within this safe range. Any integer calculations beyond this limit could lead to precision loss and incorrect results.

`BigInt` solves this problem by allowing operations on arbitrarily large integers. A `BigInt` is created by appending `n` to the end of an integer literal, or by calling the `BigInt` constructor.

**Key characteristics of JavaScript `BigInt`:**

- **Primitive Value:** `BigInt` is one of the seven primitive data types.
- **Arbitrary Precision:** Can represent integers of any size, limited only by available memory.
- **Distinct from `number`:** `BigInt` values are not strictly equal to `number` values, even if they represent the same numerical value (e.g., `1n !== 1`).
- **Type Coercion:** Operations mixing `BigInt` and `number` types are generally not allowed and will throw a `TypeError`, promoting type safety and preventing accidental precision loss. You must explicitly convert one type to the other.
- **No Decimal Support:** `BigInt` is exclusively for whole numbers; it does not support decimal places.

# **Related Concepts:**

---
- **`number` data type:** The standard JavaScript numeric type, limited by 253−1 for safe integer operations.
- **IEEE 754:** The standard for floating-point arithmetic that JavaScript's `number` type adheres to, explaining its precision limitations.
- **`Number.MAX_SAFE_INTEGER` and `Number.MIN_SAFE_INTEGER`:** Constants representing the boundaries of safe integer representation for the `number` type.
- **Type Coercion:** The strict type checking between `BigInt` and `number` is a key aspect, requiring explicit conversions.
- **Arithmetic Operations:** All standard arithmetic operators (`+`, `-`, `*`, `/`, `%`, `**`) work with `BigInt`, but only with other `BigInt` operands. Division (`/`) with `BigInt` truncates any fractional part, returning an integer `BigInt`.
- **Comparison Operators:** Comparison operators (`<`, `>`, `<=`, `>=`, `\==`, `\===`) work between `BigInt` and `number`, although strict equality (`\===`) will always be `false` if types differ.

# **Examples:**

---
```Javascript
//
---
Creating BigInts
---
// 1. Appending 'n' to an integer literal
const largeNumber = 9007199254740991n; // This is Number.MAX_SAFE_INTEGER
const reallyLargeNumber = 90071992547409912345678901234567890n;
console.log(reallyLargeNumber); // Output: 90071992547409912345678901234567890n
console.log(typeof reallyLargeNumber); // Output: bigint

// 2. Using the BigInt constructor for numbers or strings
const anotherBigInt = BigInt(123); // From a number
console.log(anotherBigInt); // Output: 123n

const stringBigInt = BigInt("98765432109876543210"); // From a string
console.log(stringBigInt); // Output: 98765432109876543210n

//
---

Comparing BigInt with Number
---
const numValue = 123;
const bigintValue = 123n;

console.log(numValue === bigintValue); // Output: false (Different types)
console.log(numValue == bigintValue); // Output: true (Loose equality allows coercion)

//
---

Arithmetic Operations with BigInt
---
const a = 10n;
const b = 5n;

console.log(a + b); // Addition: 15n
console.log(a - b); // Subtraction: 5n
console.log(a * b); // Multiplication: 50n
console.log(a / b); // Division: 2n (result is always an integer, truncates decimals)
console.log(10n / 3n); // Output: 3n (2... is truncated to 2)
console.log(a % b); // Modulus: 0n
console.log(a ** 2n); // Exponentiation: 100n

//
---

Type Coercion Restrictions (Important!)
---
const smallNum = 10;
const bigNum = 5n;

// console.log(smallNum + bigNum); // Throws TypeError: Cannot mix BigInt and other types, use explicit conversions
// console.log(smallNum * bigNum); // Throws TypeError

// Explicit conversions are necessary:
console.log(BigInt(smallNum) + bigNum); // Converts smallNum to BigInt: 10n + 5n = 15n
console.log(smallNum + Number(bigNum)); // Converts bigNum to Number: 10 + 5 = 15

//
---

Comparison Operators (mixed types allowed)
---
// Note: While arithmetic operations don't allow mixing, comparisons do.
console.log(10n > 5); // Output: true
console.log(10n < 5); // Output: false
console.log(5n === 5); // Output: false (strict equality)
console.log(5n == 5); // Output: true (loose equality)

//
---

Using BigInt with `Number.MAX_SAFE_INTEGER`
---
const maxSafe = Number.MAX_SAFE_INTEGER;
const maxSafeBigInt = BigInt(maxSafe);
const beyondMaxSafe = maxSafe + 1; // This number will lose precision as a 'number'
const beyondMaxSafeBigInt = maxSafeBigInt + 1n; // This remains precise as a 'BigInt'

console.log("Max Safe Integer (number):", maxSafe); // 9007199254740991
console.log("Beyond Max Safe (number):", beyondMaxSafe); // 9007199254740992 (accurate by chance for +1)
console.log(beyondMaxSafe === maxSafe + 2); // This might be true for other values due to loss

console.log("Max Safe Integer (BigInt):", maxSafeBigInt); // 9007199254740991n
console.log("Beyond Max Safe (BigInt):", beyondMaxSafeBigInt); // 9007199254740992n

// Demonstrate precision loss with regular numbers vs BigInt
let num1 = 9007199254740995; // Larger than MAX_SAFE_INTEGER
let num2 = 9007199254740996;
console.log(num1); // Output: 9007199254740995 (might lose precision)
console.log(num2); // Output: 9007199254740996 (might lose precision)
console.log(num1 === num2); // Can return true unexpectedly if precision is lost for these numbers

let bigNum1 = 9007199254740995n;
let bigNum2 = 9007199254740996n;
console.log(bigNum1); // Output: 9007199254740995n (always precise)
console.log(bigNum2); // Output: 9007199254740996n (always precise)
console.log(bigNum1 === bigNum2); // Output: false (always correct)

//
---

Conditional (Boolean) Context
---
// BigInts are truthy except for 0n
console.log(Boolean(1n)); // Output: true
console.log(Boolean(0n)); // Output: false
```

# **Flashcards:**

---
What problem does the `BigInt` data type solve in JavaScript?;; `BigInt` allows JavaScript to represent and perform arithmetic operations on **arbitrarily large integers**, overcoming the 253−1 safe integer limit of the standard `number` type.

How do you create a `BigInt` literal in JavaScript?;; By **appending `n`** to the end of an integer literal (e.g., `123n`), or by using the `BigInt` constructor (e.g., `BigInt("123")`).

Can you mix `BigInt` and `number` types directly in arithmetic operations?;; **No**, you cannot. Attempting to do so will result in a `TypeError`. You must explicitly convert one type to the other before performing the operation.