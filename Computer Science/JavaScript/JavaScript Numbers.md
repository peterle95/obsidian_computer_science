---
memory: done
tags:
  - mastered
language:
  - JavaScript
review-date: ""
last-reviewed: 2025-08-24
scheda: done
visit-count: 4
confidence-level: 3
consecutive-correct: 4
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
In JavaScript, the `number` primitive data type is used to represent both integer and floating-point numbers. Unlike some other programming languages that have separate types for integers and floats (e.g., `int` and `float` or `double`), JavaScript uses a single `number` type for all numeric values. Internally, JavaScript numbers are represented as 64-bit floating-point numbers, following the IEEE 754 standard (also known as "double precision").

This single representation means that even what appears to be an integer (e.g., `5`) is internally treated as a floating-point number (e.g., `5.0`). This can sometimes lead to small precision issues with very large or very small decimal numbers, common in floating-point arithmetic.

**Key characteristics of JavaScript Numbers:**

- **Primitive Value:** `number` is one of the seven primitive data types.
- **Double-Precision Floating-Point:** All numbers are stored as 64-bit floating-point values, which allows for a wide range of values but can introduce precision limitations for extremely large integers or precise decimal calculations.
- **Special Numeric Values:** JavaScript includes special global number values:
 - `Infinity`: Represents positive infinity (e.g., `1 / 0`).
 - `-Infinity`: Represents negative infinity (e.g., `-1 / 0`).
 - `NaN` (Not-a-Number): Represents a value that is not a legal number (e.g., `0 / 0`, `parseInt("hello")`). It's important to note that `NaN` is unique in that `NaN === NaN` evaluates to `false`. You should use `isNaN` or `Number.isNaN` to check for `NaN`.
- **`Number` Object Wrapper:** Similar to `String` objects, there's a `Number` object wrapper (`new Number`). However, using primitive number values is almost always preferred for consistency and performance.

# **Related Concepts:**

---
- **`BigInt`:** A newer primitive data type (ES11) introduced to handle arbitrarily large integers, beyond the safe integer limit of the `number` type (253−1).
- **Floating-Point Precision:** The inherent limitations of floating-point representation, which can lead to situations where 0.1+0 does not exactly equal 0.
- **`Math` Object:** A built-in object that provides properties and methods for mathematical constants and functions (e.g., `Math.PI`, `Math.round`, `Math.random`).
- **Type Coercion:** JavaScript's automatic conversion of values to numbers in certain contexts (e.g., arithmetic operations).
- **`parseInt` and `parseFloat`:** Global functions used to parse a string argument and return an integer or a floating-point number, respectively.
- **`Number` constructor/function:** Can be used as a function to convert a value to a number, or as a constructor (`new Number`) to create a `Number` object.
- **`isFinite` and `isNaN` (global functions) / `Number.isFinite` and `Number.isNaN` (static methods):** Functions to reliably check if a value is a finite number or `NaN`. `Number.isFinite` and `Number.isNaN` are generally preferred as they don't coerce the input.

# **Examples:**

---
```Javascript
//
---
Basic Number Literals
---
// Integer
let integerNumber = 42;
console.log(integerNumber); // Output: 42
console.log(typeof integerNumber); // Output: number

// Floating-point number
let floatNumber = 3.14159;
console.log(floatNumber); // Output: 3
console.log(typeof floatNumber); // Output: number

// Numbers with exponents (scientific notation)
let bigNumber = 1.23e6; // 1 * 10^6 = 1230000
console.log(bigNumber); // Output: 1230000
let smallNumber = 5e-3; // 5 * 10^-3 = 0
console.log(smallNumber); // Output: 0

// Hexadecimal, Octal, Binary (ES6+ for Octal/Binary literals)
let hex = 0xFF; // 255 in decimal
let octal = 0o377; // 255 in decimal (prefix 0o)
let binary = 0b11111111; // 255 in decimal (prefix 0b)
console.log(hex); // Output: 255
console.log(octal); // Output: 255
console.log(binary); // Output: 255

//
---

Special Numeric Values
---
// Infinity
let positiveInfinity = 1 / 0;
console.log(positiveInfinity); // Output: Infinity
console.log(typeof positiveInfinity); // Output: number

let negativeInfinity = -1 / 0;
console.log(negativeInfinity); // Output: -Infinity

// NaN (Not-a-Number)
let notANumber = 0 / 0;
console.log(notANumber); // Output: NaN
console.log(typeof notANumber); // Output: number

let parsedNaN = parseInt("hello world");
console.log(parsedNaN); // Output: NaN

// Checking for NaN (IMPORTANT: NaN === NaN is false!)
console.log(notANumber === NaN); // Output: false (This is a unique property of NaN)
console.log(isNaN(notANumber)); // Output: true (Global isNaN - coerces to number)
console.log(Number.isNaN(notANumber)); // Output: true (Number.isNaN - checks strictly for NaN)

//
---

Floating-Point Precision Issues
---
// This is a common demonstration of floating-point arithmetic quirks
let sum = 0 + 0.2;
console.log(sum); // Output: 0 (not exactly 0.3)
console.log(sum === 0.3); // Output: false

// How to handle precision for comparison (e.g., financial calculations)
// You often need to use a small epsilon or round to a certain number of decimal places
let epsilon = Number.EPSILON; // A very small number representing the difference between 1 and the smallest floating point number greater than 1
console.log(Math.abs(sum - 0.3) < epsilon); // Output: true (a more robust comparison)

//
---

Number Methods and Conversions
---
// toString: Converts a number to a string
let num = 123;
let numString = num.toString;
console.log(numString); // Output: "123"
console.log(typeof numString); // Output: string

// toFixed(digits): Formats a number to a specified number of decimal places (returns a string)
let price = 12.34567;
let fixedPrice = price.toFixed(2); // Rounds to 2 decimal places
console.log(fixedPrice); // Output: "12.35"
console.log(typeof fixedPrice); // Output: string

// toPrecision(precision): Formats a number to a specified length
let value = 123.456;
console.log(value.toPrecision(4)); // Output: "123.5" (4 significant digits)

// parseInt and parseFloat - for converting strings to numbers
let strInt = "100px";
let strFloat = "3.14rem";
let strMixed = "abc123";

console.log(parseInt(strInt)); // Output: 100 (parses until a non-numeric character)
console.log(parseFloat(strFloat)); // Output: 3
console.log(parseInt(strMixed)); // Output: NaN

// Number constructor/function - to convert a value to a number
console.log(Number("123")); // Output: 123
console.log(Number("3.14")); // Output: 3
console.log(Number(true)); // Output: 1
console.log(Number(false)); // Output: 0
console.log(Number("")); // Output: 0
console.log(Number(" 123 ")); // Output: 123 (trims whitespace)
console.log(Number("hello")); // Output: NaN

//
---

Number's Safe Integer Limits
---
// JavaScript can safely represent integers between -(2^53 - 1) and 2^53 - 1
console.log(Number.MAX_SAFE_INTEGER); // Output: 9007199254740991
console.log(Number.MIN_SAFE_INTEGER); // Output: -9007199254740991

// Numbers beyond this limit might lose precision
console.log(9007199254740992 === 9007199254740993); // Output: true (incorrect, due to precision loss)

// Use BigInt for numbers larger than MAX_SAFE_INTEGER
const veryBigNum = 9007199254740992n; // Appending 'n' makes it a BigInt
console.log(typeof veryBigNum); // Output: bigint
```

# **Flashcards:**

---
What is the single data type in JavaScript used for both integers and floating-point numbers?;; The `number` primitive data type.

Name the three special numeric values in JavaScript.;; `Infinity`, `-Infinity`, and `NaN` (Not-a-Number).

How do you reliably check if a value is `NaN` in JavaScript, and why is `NaN === NaN` unreliable?;; Use `isNaN` or, preferably, `Number.isNaN`. `NaN === NaN` evaluates to `false` because `NaN` is unique and never equal to itself.