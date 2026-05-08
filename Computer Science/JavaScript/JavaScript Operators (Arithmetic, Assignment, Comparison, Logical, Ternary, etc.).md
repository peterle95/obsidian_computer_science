---
memory: done
tags:
  - mastered
language:
  - JavaScript
review-date: ""
last-reviewed: 2025-09-25
scheda: done
visit-count: 2
confidence-level: 1.5
consecutive-correct: 2
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
JavaScript operators are the building blocks for performing computations, making decisions, and manipulating data. They solve the fundamental problem of how to interact with and transform values within a program. Without operators, a programming language would be largely static, unable to process input, perform calculations, or control program flow based on conditions.

Their primary applications include:

- **Mathematical calculations:** Performing addition, subtraction, multiplication, division, etc.
- **Assigning values:** Storing the result of an operation or a literal value into a variable.
- **Comparing values:** Determining relationships between values (e.g., greater than, equal to, not equal to), which is crucial for conditional logic.
- **Controlling program flow:** Combining conditions to create complex decision-making structures (e.g., `if` statements, loops).
- **Manipulating data types:** Performing operations like string concatenation or checking the type of a variable.

Operators are fundamental and important in computer science and JavaScript specifically because they enable:

- **Computation:** The ability to process data and derive new information.
- **Logic and Control Flow:** The means to make programs dynamic and responsive by allowing different code paths based on conditions.
- **Expressiveness:** They allow developers to write concise and readable code for common operations.

# **Core Explanation:**

---
**Definition:** In JavaScript, an operator is a special symbol or keyword that performs an operation on one or more values (operands) and produces a result.

**Key Characteristics:**

- **Operands:** Operators work on operands. Depending on the number of operands, operators can be:
 - **Unary:** Operate on a single operand (e.g., `++a`, `-b`, `!c`, `typeof d`).
 - **Binary:** Operate on two operands (e.g., `a + b`, `x === y`).
 - **Ternary:** Operate on three operands (e.g., `condition ? expr1 : expr2`).
- **Return Value:** Every operator produces a result. The data type of the result depends on the operator and the types of its operands.
- **Operator Precedence:** Determines the order in which operators are evaluated in an expression (e.g., multiplication and division usually take precedence over addition and subtraction).
- **Operator Associativity:** Determines the order of evaluation for operators of the same precedence (e.g., left-to-right or right-to-left).

**How it Works (Categories of Operators):**

1. **Arithmetic Operators:** Used to perform mathematical calculations.

 - `+` (Addition)
 - `-` (Subtraction)
 - `*` (Multiplication)
 - `/` (Division)
 - `%` (Modulo - remainder of division)
 - `**` (Exponentiation - ES2016)
 - `++` (Increment - unary, adds 1)
 - `--` (Decrement - unary, subtracts 1)
2. **Assignment Operators:** Used to assign values to variables. The simple assignment operator is `/=`. Compound assignment operators perform an operation and then assign the result.

 - `/=` (Assignment)
 - `+=` (Add and assign)
 - `-=` (Subtract and assign)
 - `*=` (Multiply and assign)
 - `/=` (Divide and assign)
 - `%=` (Modulo and assign)
 - `**=` (Exponentiate and assign)
3. **Comparison Operators:** Used to compare two values and return a boolean (`true` or `false`) result.
 - `/==` (Loose equality - compares values after type coercion)
 - `!=` (Loose inequality - compares values after type coercion)
 - `/===` (Strict equality - compares values and types, no type coercion)
 - `!==` (Strict inequality - compares values and types, no type coercion)
 - `>` (Greater than)
 - `<` (Less than)
 - `>=` (Greater than or equal to)
 - `<=` (Less than or equal to)
4. **Logical Operators:** Used to combine or invert boolean expressions. They often work with "truthy" and "falsy" values.

 - `&&` (Logical AND - returns the first falsy operand or the last operand if all are truthy)
 - `||` (Logical OR - returns the first truthy operand or the last operand if all are2 falsy)
 - `!` (Logical NOT - unary, inverts the boolean value of its operand)
5. **Ternary (Conditional) Operator:** The only ternary operator in JavaScript. It provides a concise way to write a conditional expression.

 - `condition ? exprIfTrue : exprIfFalse`
6. **Other Important Operators:**

 - **`typeof` operator:** Unary, returns a string indicating the data type of its operand.
 - **[[instanceof]] operator:** Checks if an object is an instance of a particular class or constructor function.
 - **`in` operator:** Checks if a property exists in an object.
 - **Comma operator (`,`)**: Evaluates each of its operands (from left to right) and returns the value of the last operand.
 - **[[JavaScript - Nullish Coalescing Operator (？？)]] (`??`) (ES2020):** Returns its right-hand side operand when its left-hand side operand is `null` or `undefined`, otherwise returns its left-hand side operand
 - **Optional Chaining (`?.`) (ES2020):** Allows you to read the value of a property located deep within a chain of connected objects without having to explicitly validate that each reference in the chain is valid

# **Related Concepts:**

---
- **Expressions:** An expression is any valid unit of code that resolves to a value. Operators are key components of expressions, as they define how values are combined or transformed to produce a result. For example, `a + b` is an expression where `+` is an operator.

- **Statements:** A statement is an instruction that performs an action. While expressions _produce_ values, statements _do_ something. Often, expressions (which heavily use operators) are part of statements (e.g., `let result = a + b;` is an assignment statement containing an arithmetic expression).

- **Data Types (Primitives & Objects):** Operators often behave differently based on the data types of their operands (e.g., `+` performs addition for numbers but concatenation for strings). Understanding data types is crucial for predicting the outcome of an operation and avoiding unexpected type coercion.

- **Type Coercion:** JavaScript is a dynamically typed language, and it often performs automatic type conversion (coercion) when operators are used with operands of different types (e.g., `'5' + 2` results in `'52'` due to string concatenation, not numeric addition). This is a critical concept to understand when using comparison operators (`/==` vs. `/===`) and arithmetic operators.

- **Truthy and Falsy Values:** Logical operators (`&&`, `||`, `!`) often evaluate expressions based on their "truthiness" or "falsiness." In JavaScript, values that are not strictly `true` or `false` can still behave as such in a boolean context (e.g., `0`, `null`, `undefined`, `''`, `NaN` are falsy; all other values are truthy).

- **Operator Precedence and Associativity:** These rules dictate the order in which operators in a complex expression are evaluated. They are fundamental to ensuring that expressions are parsed and computed as intended. For example, `2 + 3 * 4` evaluates to `14` (multiplication before addition) due to precedence. Parentheses `` can override default precedence.

- **Control Flow (Conditional Statements, Loops):** Comparison and logical operators are indispensable for control flow statements like `if/else`, `switch`, `for`, and `while` loops. They define the conditions under which blocks of code are executed or repeated.

# **Examples:**

---
```js
// 1. Arithmetic Operators
console.log("
---
Arithmetic Operators
---
");
let num1 = 10;
let num2 = 3;

// Addition (+)
let sum = num1 + num2; // 10 + 3 = 13
console.log("Sum:", sum);

// Subtraction (-)
let difference = num1 - num2; // 10 - 3 = 7
console.log("Difference:", difference);

// Multiplication (*)
let product = num1 * num2; // 10 * 3 = 30
console.log("Product:", product);

// Division (/)
let quotient = num1 / num2; // 10 / 3 = 3...
console.log("Quotient:", quotient);

// Modulo (%) - returns the remainder of the division
let remainder = num1 % num2; // 10 % 3 = 1 (because 10 = 3*3 + 1)
console.log("Remainder:", remainder);

// Exponentiation (**) - ES2016
let power = num1 ** num2; // 10 to the power of 3 (10 * 10 * 10) = 1000
console.log("Power:", power);

// Increment (++) - adds 1 to the operand
let count = 5;
count++; // Post-increment: uses the value then increments. count is now 6
console.log("Incremented count (post):", count); // 6

let preIncrement = ++count; // Pre-increment: increments then uses the value. count is now 7, preIncrement is 7
console.log("Pre-incremented count:", preIncrement); // 7
console.log("Count after pre-increment:", count); // 7

// Decrement (--) - subtracts 1 from the operand
let value = 8;
value--; // Post-decrement: uses the value then decrements. value is now 7
console.log("Decremented value (post):", value); // 7

let preDecrement = --value; // Pre-decrement: decrements then uses the value. value is now 6, preDecrement is 6
console.log("Pre-decremented value:", preDecrement); // 6
console.log("Value after pre-decrement:", value); // 6

// Type Coercion with + operator (string concatenation vs. addition)
console.log("String concatenation with +:");
console.log("Hello" + " " + "World"); // "Hello World" (string concatenation)
console.log("Number" + 10); // "Number10" (number 10 is coerced to string "10")
console.log(5 + "2"); // "52" (number 5 is coerced to string "5")

// 2. Assignment Operators
console.log("\n
---

Assignment Operators
---
");
let x = 10; // Simple assignment: assigns 10 to x
console.log("Initial x:", x);

x += 5; // x = x + 5; // x is now 15
console.log("x after x += 5:", x);

x -= 3; // x = x - 3; // x is now 12
console.log("x after x -= 3:", x);

x *= 2; // x = x * 2; // x is now 24
console.log("x after x *= 2:", x);

x /= 4; // x = x / 4; // x is now 6
console.log("x after x /= 4:", x);

x %= 5; // x = x % 5; // x is now 1 (remainder of 6 / 5)
console.log("x after x %= 5:", x);

let y = 2;
y **= 3; // y = y ** 3; // y is now 8 (2 * 2 * 2)
console.log("y after y **= 3:", y);

// 3. Comparison Operators
console.log("\n
---

Comparison Operators
---
");
let a = 5;
let b = "5"; // String type
let c = 10;

// Loose Equality (==): Compares values after type coercion
console.log("a == b:", a == b); // true (5 == "5" -> "5" is coerced to 5)
console.log("a == c:", a == c); // false

// Strict Equality (===): Compares values AND types (no type coercion)
console.log("a === b:", a === b); // false (number 5 is not strictly equal to string "5")
console.log("a === 5:", a === 5); // true

// Loose Inequality (!=): True if values are not equal after coercion
console.log("a != b:", a != b); // false (5 != "5" is false because they are loosely equal)

// Strict Inequality (!==): True if values OR types are not equal
console.log("a !== b:", a !== b); // true (number 5 is not strictly equal to string "5")
console.log("a !== 10:", a !== 10); // true

// Greater than (>)
console.log("a > c:", a > c); // false (5 > 10)
console.log("c > a:", c > a); // true (10 > 5)

// Less than (<)
console.log("a < c:", a < c); // true (5 < 10)

// Greater than or equal to (>=)
console.log("a >= 5:", a >= 5); // true
console.log("c >= a:", c >= a); // true

// Less than or equal to (<=)
console.log("a <= 5:", a <= 5); // true
console.log("a <= c:", a <= c); // true

// 4. Logical Operators
console.log("\n
---

Logical Operators
---
");
let age = 25;
let hasLicense = true;
let isStudent = false;

// Logical AND (&&): Returns true if both operands are truthy
// Short-circuits: If the first operand is falsy, it returns that operand immediately.
console.log("age > 18 && hasLicense:", age > 18 && hasLicense); // (true && true) -> true
console.log("age < 18 && hasLicense:", age < 18 && hasLicense); // (false && true) -> false (and stops evaluation)

let name = "Alice";
let city = "";
// Returns the first falsy value or the last value if all are truthy
console.log(name && city); // "" (city is falsy, so it's returned)
console.log(name && age); // 25 (both truthy, so the last value is returned)

// Logical OR (||): Returns true if at least one operand is truthy
// Short-circuits: If the first operand is truthy, it returns that operand immediately.
console.log("age > 30 || hasLicense:", age > 30 || hasLicense); // (false || true) -> true
console.log("isStudent || age < 20:", isStudent || age < 20); // (false || false) -> false

let defaultName = "";
let userName = "Bob";
// Returns the first truthy value or the last value if all are falsy
console.log(defaultName || userName); // "Bob" (userName is truthy, so it's returned)
console.log(null || undefined); // undefined (both falsy, last one returned)

// Logical NOT (!): Inverts the boolean value of its operand
console.log("!hasLicense:", !hasLicense); // !true -> false
console.log("!isStudent:", !isStudent); // !false -> true
console.log("!0:", !0); // !falsy -> true
console.log("!\"\":", !""); // !falsy -> true

// 5. Ternary (Conditional) Operator
console.log("\n
---

Ternary Operator
---
");
let temperature = 28;
let weatherMessage = (temperature > 25) ? "It's hot outside!" : "It's not too hot.";
// If temperature > 25 is true, weatherMessage gets "It's hot outside!".
// Otherwise, weatherMessage gets "It's not too hot.".
console.log(weatherMessage); // "It's hot outside!"

let status = "active";
let color = (status === "active") ? "green" : "red";
console.log("Status color:", color); // "green"

// 6. Other Important Operators

// typeof operator: Returns a string indicating the data type of an operand
console.log("\n
---
typeof Operator
---
");
console.log("Type of num1:", typeof num1); // "number"
console.log("Type of hasLicense:", typeof hasLicense); // "boolean"
console.log("Type of 'hello':", typeof "hello"); // "string"
console.log("Type of :", typeof ); // "object" (arrays are objects in JS)
console.log("Type of {}:", typeof {}); // "object"
console.log("Type of null:", typeof null); // "object" (a well-known, historic bug in JS)
console.log("Type of undefined:", typeof undefined); // "undefined"
console.log("Type of function {}:", typeof function {}); // "function"

// Nullish Coalescing Operator (??) - ES2020
console.log("\n
---

Nullish Coalescing Operator (??)
---
");
let userSetting = null;
let defaultSetting = "Dark Mode";
// Returns defaultSetting if userSetting is null or undefined; otherwise, returns userSetting.
let finalSetting = userSetting ?? defaultSetting;
console.log("Final setting (null userSetting):", finalSetting); // "Dark Mode"

userSetting = "Light Mode";
finalSetting = userSetting ?? defaultSetting;
console.log("Final setting (defined userSetting):", finalSetting); // "Light Mode"

userSetting = 0; // 0 is a falsy value but not null/undefined
finalSetting = userSetting ?? defaultSetting;
console.log("Final setting (0 userSetting):", finalSetting); // 0 (?? treats 0 as a valid value)

// Optional Chaining (?.) - ES2020
console.log("\n
---

Optional Chaining (?.)
---
");
const user = {
 profile: {
 address: {
 street: "123 Main St"
 }
 },
 company: null // Company is null
};

// Without optional chaining, accessing non-existent nested properties would throw an error.
// console.log(user.preferences.theme); // Throws TypeError: Cannot read properties of undefined (reading 'theme')

// With optional chaining, it returns undefined if any part of the chain is null or undefined.
console.log("User street:", user.profile?.address?.street); // "123 Main St"
console.log("User theme:", user.preferences?.theme); // undefined (no error)
console.log("User company name:", user.company?.name); // undefined (no error)

// Calling methods with optional chaining
const admin = {
 getName { return "Admin User"; }
};

const guest = {};

console.log(admin.getName?.); // "Admin User" (method exists and is called)
console.log(guest.getName?.); // undefined (method does not exist, no error)
```

# **Flashcards:**

---
What is the purpose of operators in JavaScript?;;To perform computations, make decisions, and manipulate values within a program.

What is the difference between `/==` and `/===` in JavaScript?;;`/==` (loose equality) compares values after type coercion, while `/===` (strict equality) compares values AND types without coercion.

Name three categories of JavaScript operators and give an example for each.;;Arithmetic (`+`), Assignment (`/=`), Comparison (`>`), Logical (`&&`), Ternary (`? :`), etc. (any three categories with correct examples).