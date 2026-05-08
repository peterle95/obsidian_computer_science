---
memory: to_finish
tags:
  - learned
language:
  - JavaScript
review-date:
last-reviewed: 2025-10-09
scheda: done
visit-count: 6
confidence-level: 2.5
consecutive-correct: 4
last-struggle-date: 2025-08-19
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
# **Purpose/Why**:
---
JavaScript array destructuring solves the fundamental problem of <mark style="background: #ABF7F7A6;">extracting values from arrays in a clean, readable way without repetitive bracket notation and index access</mark>. It eliminates verbose code like `const first = array[0]; const second = array[1];` and replaces it with <mark style="background: #ABF7F7A6;">concise syntax that clearly expresses intent</mark>. This ES6 feature is crucial for modern JavaScript development because it makes code more maintainable, reduces errors from incorrect indexing, enables elegant pattern matching, and provides powerful features like swapping variables, handling function returns, and working with nested structures in a declarative manner.*

# **Core Explanation:**
---
## *Array destructuring is a JavaScript ES6 syntax that allows unpacking values from arrays into distinct variables using a pattern-matching approach:*

**Basic Syntax:**
- **Pattern**: `const [variable1, variable2, ...] = array`
- **Position-based**: Variables are assigned based on their position in the destructuring pattern
- **Declaration types**: Works with `const`, `let`, `var`, and assignment to existing variables

**Key Features:**
- **Skipping elements**: <mark style="background: #FFB86CA6;">Use empty slots to skip array positions</mark>
- **Default values**: Assign fallback values when array elements are undefined
- **Rest operator**: Collect remaining elements into a new array using `...`
- **Nested destructuring**: Extract values from nested arrays
- **Swapping**: Exchange variable values without temporary variables

**Advanced Patterns:**
- **Function parameters**: Destructure arrays passed to functions
- **Mixed with objects**: Combine array and object destructuring
- **Iterables**: Works with any iterable, not just arrays

*Key characteristics: Assignment happens left-to-right based on position, missing array elements result in undefined unless default values are provided, it's syntactic sugar that doesn't modify the original array, and it supports complex nested patterns.*

# **Related Concepts:**
---
## *Array destructuring connects to several important programming concepts:*

- **Pattern Matching**: Declarative way to extract data based on structure
- **Object Destructuring**: Similar ES6 feature for extracting object properties
- **Spread/Rest Operator**: Using `...` to collect or spread array elements
- **ES6 Features**: Part of modern JavaScript syntax improvements
- **Multiple Assignment**: Assigning multiple variables in one statement
- **Functional Programming**: Elegant data extraction for functional patterns
- **Iterable Protocol**: Works with any object that implements Symbol.iterator
- **Parameter Destructuring**: Using destructuring in function parameter lists

# **Examples:**
---
```javascript
// BASIC ARRAY DESTRUCTURING

console.log('=== Basic Destructuring ===');
const colors = ['red', 'green', 'blue', 'yellow'];

// Traditional way (verbose)
const firstColorOld = colors[0];
const secondColorOld = colors[1];
console.log('Old way:', firstColorOld, secondColorOld);

// Destructuring way (clean and clear)
const [firstColor, secondColor, thirdColor] = colors;
console.log('Destructured:', firstColor, secondColor, thirdColor);
// Output: red green blue

// Variables are assigned based on position, not name
const [primary, secondary, tertiary] = colors;
console.log('Named by position:', primary, secondary, tertiary);
// Output: red green blue

// SKIPPING ELEMENTS

console.log('=== Skipping Elements ===');
const numbers = [1, 2, 3, 4, 5];

// Skip elements by leaving empty slots
const [first, , third, , fifth] = numbers;
console.log('Skipped elements:', first, third, fifth);  // 1 3 5

// Skip first few elements to get the end
const [, , , fourthNumber, fifthNumber] = numbers;
console.log('Last two:', fourthNumber, fifthNumber);  // 4 5

// DEFAULT VALUES

console.log('=== Default Values ===');
const incompleteArray = ['apple', 'banana'];

// Without defaults - undefined for missing elements
const [fruit1, fruit2, fruit3] = incompleteArray;
console.log('Without defaults:', fruit1, fruit2, fruit3);
// Output: apple banana undefined

// With default values - fallback when element is undefined
const [item1, item2, item3 = 'orange', item4 = 'grape'] = incompleteArray;
console.log('With defaults:', item1, item2, item3, item4);
// Output: apple banana orange grape

// Defaults only used when value is undefined (not for null or falsy values)
const arrayWithNull = ['first', null, undefined];
const [a = 'default-a', b = 'default-b', c = 'default-c'] = arrayWithNull;
console.log('Null vs undefined:', a, b, c);
// Output: first null default-c (only undefined gets default)

// REST OPERATOR (...) - Collecting remaining elements

console.log('=== Rest Operator ===');
const alphabet = ['a', 'b', 'c', 'd', 'e', 'f'];

// Get first element and collect rest
const [firstLetter, ...restLetters] = alphabet;
console.log('First:', firstLetter);        // 'a'
console.log('Rest:', restLetters);         // ['b', 'c', 'd', 'e', 'f']

// Get first two and collect rest
const [letter1, letter2, ...remaining] = alphabet;
console.log('First two:', letter1, letter2);  // 'a' 'b'
console.log('Remaining:', remaining);          // ['c', 'd', 'e', 'f']

// Rest must be last element in destructuring pattern
// const [first, ...middle, last] = array;  // SyntaxError!

// SWAPPING VARIABLES

console.log('=== Variable Swapping ===');
let x = 10;
let y = 20;
console.log('Before swap:', x, y);  // 10 20

// Traditional swapping requires temporary variable
// let temp = x; x = y; y = temp;

// Destructuring swap (elegant and concise)
[x, y] = [y, x];
console.log('After swap:', x, y);   // 20 10

// Multiple variable swapping
let a = 1, b = 2, c = 3;
[a, b, c] = [c, a, b];  // Rotate values
console.log('Rotated:', a, b, c);   // 3 1 2

// NESTED ARRAY DESTRUCTURING

console.log('=== Nested Destructuring ===');
const nestedArray = [
  ['John', 'Doe'],
  ['Jane', 'Smith'],
  [25, 30]
];

// Destructure nested arrays
const [[firstName1, lastName1], [firstName2, lastName2], [age1, age2]] = nestedArray;
console.log('Person 1:', firstName1, lastName1, 'Age group includes:', age1);
console.log('Person 2:', firstName2, lastName2, 'Age group includes:', age2);

// Mixed destructuring with skipping
const [[first], , [youngAge]] = nestedArray;
console.log('First name and young age:', first, youngAge);  // John 25

// Deep nesting example
const deepNested = [1, [2, [3, [4, 5]]]];
const [level1, [level2, [level3, [level4, level5]]]] = deepNested;
console.log('Deep levels:', level1, level2, level3, level4, level5);  // 1 2 3 4 5

// FUNCTION PARAMETER DESTRUCTURING

console.log('=== Function Parameters ===');

// Function that expects an array and destructures it
function processCoordinates([x, y, z = 0]) {  // z defaults to 0 if not provided
  console.log(`Processing point: x=${x}, y=${y}, z=${z}`);
  return x + y + z;
}

const point2D = [10, 20];
const point3D = [5, 15, 25];

console.log('2D result:', processCoordinates(point2D));  // z defaults to 0
console.log('3D result:', processCoordinates(point3D));

// Function returning multiple values via array
function getStats(numbers) {
  const sum = numbers.reduce((a, b) => a + b, 0);
  const avg = sum / numbers.length;
  const max = Math.max(...numbers);
  const min = Math.min(...numbers);
  
  return [sum, avg, max, min];  // Return array of results
}

// Destructure the returned array
const [total, average, maximum, minimum] = getStats([1, 2, 3, 4, 5]);
console.log(`Stats: Total=${total}, Avg=${average}, Max=${maximum}, Min=${minimum}`);

// WORKING WITH STRINGS (strings are iterable)

console.log('=== String Destructuring ===');
const word = 'HELLO';

// Destructure string into individual characters
const [char1, char2, char3, char4, char5] = word;
console.log('Characters:', char1, char2, char3, char4, char5);  // H E L L O

// Get first and last characters
const [firstChar, ...middleChars] = word;
const lastChar = middleChars.pop();  // Remove and get last element
console.log('First:', firstChar, 'Last:', lastChar);  // H O

// PRACTICAL EXAMPLES

console.log('=== Practical Use Cases ===');

// 1. Processing CSV-like data
const csvRow = 'John,Doe,30,Engineer,New York';
const [firstName, lastName, age, job, city] = csvRow.split(',');
console.log(`Employee: ${firstName} ${lastName}, ${age} years old, ${job} in ${city}`);

// 2. Working with regular expression matches
const email = 'user@example.com';
const emailRegex = /^([^@]+)@([^@]+\.[^@]+)$/;
const match = email.match(emailRegex);

if (match) {
  const [fullMatch, username, domain] = match;
  console.log(`Email parts: username="${username}", domain="${domain}"`);
}

// 3. Handling multiple function return values
function divideWithRemainder(dividend, divisor) {
  const quotient = Math.floor(dividend / divisor);
  const remainder = dividend % divisor;
  return [quotient, remainder];
}

const [result, leftover] = divideWithRemainder(17, 5);
console.log(`17 ÷ 5 = ${result} remainder ${leftover}`);

// 4. Array manipulation - getting head and tail
function processArray(arr) {
  if (arr.length === 0) return 'Empty array';
  
  const [head, ...tail] = arr;
  console.log(`Head: ${head}, Tail: [${tail.join(', ')}]`);
  return { head, tail };
}

processArray([1, 2, 3, 4, 5]);
processArray(['first']);
processArray([]);

// 5. Loading configuration with defaults
const config = ['localhost', 3000];  // Incomplete config
const [host = 'localhost', port = 8080, protocol = 'http'] = config;
console.log(`Server config: ${protocol}://${host}:${port}`);

// 6. Iterating with destructuring
const students = [
  ['Alice', 85],
  ['Bob', 92],
  ['Charlie', 78]
];

console.log('Student grades:');
students.forEach(([name, grade]) => {
  // Destructure each array element in the forEach callback
  const status = grade >= 80 ? 'Pass' : 'Needs improvement';
  console.log(`${name}: ${grade} (${status})`);
});

// 7. Working with Map entries
const userRoles = new Map([
  ['admin', ['read', 'write', 'delete']],
  ['user', ['read']],
  ['guest', ['read']]
]);

console.log('User permissions:');
for (const [role, permissions] of userRoles) {
  // Destructure Map entries (each entry is [key, value])
  console.log(`${role}: ${permissions.join(', ')}`);
}

// 8. Error handling with optional destructuring
function safeDivide(a, b) {
  if (b === 0) return [null, 'Division by zero'];
  return [a / b, null];
}

const [quotient1, error1] = safeDivide(10, 2);
const [quotient2, error2] = safeDivide(10, 0);

console.log(error1 ? `Error: ${error1}` : `Result: ${quotient1}`);  // Result: 5
console.log(error2 ? `Error: ${error2}` : `Result: ${quotient2}`);  // Error: Division by zero
````

# **Flashcards:**

---

What is the basic syntax for array destructuring in JavaScript?;; const [variable1, variable2, variable3] = array; Variables are assigned based on their position in the array

How do you skip elements when destructuring an array?;; Use empty slots: const [first, , third, , fifth] = array; Empty commas skip those positions

How do you provide default values in array destructuring?;; Use assignment operator: const [a = 'default', b = 'fallback'] = array; Defaults are used only when the array element is undefined

What does the rest operator (...) do in array destructuring?;; It collects remaining elements into a new array: const [first, ...rest] = array; Must be the last element in the destructuring pattern

How do you swap two variables using array destructuring?;; [x, y] = [y, x]; This swaps the values without needing a temporary variable

How do you destructure nested arrays?;; Use nested brackets: const [/[a, b], [c, d]] = [/[1, 2], [3, 4]]; Pattern matches the array structure