---
memory: to_finish
tags:
 - learned
language:
 - JavaScript
review-date: ""
last-reviewed: 2025-08-05
scheda: done
visit-count: 1
confidence-level: 1
consecutive-correct: 0
last-struggle-date: 2025-08-05

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
In JavaScript, a `string` is a primitive data type used to represent sequences of characters. Strings are fundamental for handling text-based information, from single letters to entire paragraphs. They are enclosed in single quotes (`'...'`), double quotes (`"..."`), or backticks (`...`).

**Key characteristics of JavaScript Strings:**

- **Primitive Value:** Strings are one of the seven primitive data types. This means they represent a single, simple value, not a complex collection like objects.
- **Immutability:** Once a string is created, its value cannot be changed. Any operation that appears to modify a string (e.g., `toUpperCase`, `concat`) actually returns a _new_ string with the desired modification, leaving the original string untouched. This is a crucial concept for understanding string manipulation.
- **Zero-Indexed:** Characters within a string are accessed using zero-based indexing, meaning the first character is at index `0`, the second at `1`, and so on.
- **String Primitives vs. String Objects:** While typically used as primitives, JavaScript also has a `String` object wrapper (`new String`). However, using `String` objects directly is generally discouraged due to potential confusion with primitive string behavior and performance implications. Most of the time, you'll be working with primitive strings.

# **Related Concepts:**

---
- **Template Literals (Template Strings):** Introduced in ES6, these allow for embedded expressions and multi-line strings using backticks (```). They provide a more flexible and readable way to construct strings compared to traditional concatenation.
- **Escape Sequences:** Special character combinations (e.g., `\n` for newline, `\t` for tab, `\'` for single quote) used within strings to represent characters that are difficult or impossible to type directly.
- **Unicode:** JavaScript strings are internally represented using Unicode (specifically UTF-16 encoding). This allows strings to handle characters from virtually all written languages.
- **String Methods:** JavaScript provides a rich set of built-in methods for performing various operations on strings, such as searching, replacing, extracting substrings, changing case, and more. Understanding these methods is key to effective string manipulation.
- **Regular Expressions:** Powerful patterns used for matching character combinations in strings. Many string methods (like `match`, `replace`, `search`, `split`) can work with regular expressions.

# **Examples:**
```Javascript
//
---
String Literals
---
// Single quotes
let singleQuoteString = 'Hello, JavaScript!';
console.log(singleQuoteString); // Output: Hello, JavaScript!

// Double quotes
let doubleQuoteString = "Welcome to strings.";
console.log(doubleQuoteString); // Output: Welcome to strings.

// Backticks (Template Literals) - allowing for multi-line strings and embedded expressions
let name = "Alice";
let age = 30;
let multiLineString = `
 Hello, ${name}!
 You are ${age} years old.
`;
console.log(multiLineString);
// Output:
//
// Hello, Alice!
// You are 30 years old.

//
---
String Immutability Example
---
let originalString = "immutable";
console.log("Original String:", originalString); // Output: Original String: immutable

let newString = originalString.toUpperCase; // toUpperCase returns a NEW string
console.log("New String (toUpperCase):", newString); // Output: New String (toUpperCase): IMMUTABLE
console.log("Original String after toUpperCase:", originalString); // Output: Original String after toUpperCase: immutable (original is unchanged)

// Concatenation also creates a new string
let part1 = "Good";
let part2 = "morning";
let combinedString = part1 + " " + part2;
console.log("Combined String:", combinedString); // Output: Combined String: Good morning
console.log("Original part1:", part1); // Output: Original part1: Good (part1 remains unchanged)

//
---

Accessing Characters (Zero-Indexed)
---
let word = "developer";
console.log(word); // Output: d
console.log(word); // Output: l
console.log(word[word.length - 1]); // Output: r (last character)

//
---

Common String Methods
---
// length: Returns the length of the string
let text = "JavaScript is fun";
console.log("Length:", text.length); // Output: Length: 17

// charAt(index): Returns the character at a specified index
console.log("Character at index 4:", text.charAt(4)); // Output: Character at index 4: S

// indexOf(substring): Returns the index of the first occurrence of a specified substring
console.log("Index of 'is':", text.indexOf("is")); // Output: Index of 'is': 11
console.log("Index of 'xyz':", text.indexOf("xyz")); // Output: Index of 'xyz': -1 (not found)

// includes(substring): Checks if a string contains a specified substring (returns boolean)
console.log("Includes 'Script':", text.includes("Script")); // Output: Includes 'Script': true

// slice(startIndex, endIndex): Extracts a part of a string
let sliced = text.slice(0, 10); // from index 0 up to (but not including) index 10
console.log("Sliced string:", sliced); // Output: Sliced string: JavaScript

let slicedToEnd = text.slice(11); // from index 11 to the end
console.log("Sliced to end:", slicedToEnd); // Output: Sliced to end: is fun

// replace(searchValue, replaceValue): Replaces occurrences of a substring
let replacedText = text.replace("fun", "awesome");
console.log("Replaced text:", replacedText); // Output: Replaced text: JavaScript is awesome
// Note: replace only replaces the first occurrence by default without regex 'g' flag

// split(separator): Splits a string into an array of substrings
let wordsArray = text.split(" ");
console.log("Words Array:", wordsArray); // Output: Words Array: [ 'JavaScript', 'is', 'fun' ]

let charactersArray = "hello".split("");
console.log("Characters Array:", charactersArray); // Output: Characters Array: [ 'h', 'e', 'l', 'l', 'o' ]

// toLowerCase and toUpperCase: Converts the string to lower or upper case
console.log("Lowercase:", text.toLowerCase); // Output: Lowercase: javascript is fun
console.log("Uppercase:", text.toUpperCase); // Output: Uppercase: JAVASCRIPT IS FUN

// trim: Removes whitespace from both ends of a string
let paddedString = " Hello, World! ";
console.log("Trimmed string:", paddedString.trim); // Output: Trimmed string: Hello, World!

// startsWith(prefix) and endsWith(suffix): Checks if a string starts or ends with a specified substring
console.log("Starts with 'Java':", text.startsWith("Java")); // Output: Starts with 'Java': true
console.log("Ends with 'fun':", text.endsWith("fun")); // Output: Ends with 'fun': true

// repeat(count): Returns a new string with a specified number of copies of the original string
let repeated = "abc".repeat(3);
console.log("Repeated string:", repeated); // Output: Repeated string: abcabcabc
```

# **Flashcards:**

---
What is a key characteristic of JavaScript strings regarding their mutability?;; JavaScript strings are **immutable**, meaning their value cannot be changed after creation.

Operations that seem to modify a string actually return a new string. How can you define a multi-line string or embed expressions within a string in JavaScript?;; Using **Template Literals** (also known as Template Strings), which are enclosed by backticks (\`\`\`).

Name three common built-in string methods in JavaScript.;; `length`, `charAt`, `indexOf`, `includes`, `slice`, `replace`, `split`, `toLowerCase`, `toUpperCase`, `trim`, `startsWith`, `endsWith`, `repeat`. (Any three are acceptable).