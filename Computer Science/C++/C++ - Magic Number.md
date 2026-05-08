---
memory: to_finish
tags:
 - learned
language:
 - C++
review-date: ""
last-reviewed: 2025-08-14
scheda: done
visit-count: 2
confidence-level: 1
consecutive-correct: 1
last-struggle-date: 2025-08-01

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
The concept of "magic numbers" in C++ (and programming in general) **identifies a common code smell and proposes a solution to improve code quality.** A "magic number" refers to a numeric literal (a number typed directly into the code, like `3.14159` or `1024`) whose meaning is not immediately clear from its value alone.

The fundamental problems that addressing magic numbers solves are:

1. **Readability:** Code filled with unexplained numbers is hard to understand. Without context, it's difficult for anyone reading the code (including the original author later) to grasp _why_ a particular number is being used.

2. **Maintainability:** If a magic number needs to change, it might appear in multiple places throughout the codebase. Changing every instance is tedious, error-prone, and can lead to inconsistencies if one instance is missed.

3. **Error Proneness:** Hardcoding values increases the risk of typos. If `100` should be `1000` and it's repeated, a typo in one place is likely.

4. **Reusability:** A hardcoded number often ties a piece of code to a very specific context, making it less generic and harder to reuse in other parts of the program or in different projects.


In C++, where performance and explicit control are often priorities, developers might be tempted to use literal values directly for perceived efficiency or simplicity in small cases. However, this quickly leads to unmanageable codebases. It's important in C++ to name constants clearly for the same reasons as other languages, but also because C++'s strong typing and compile-time capabilities offer excellent ways to define and use these constants efficiently without runtime overhead.

# **Core Explanation:**

---
==A **magic number** in programming is a numeric literal (a constant value) that appears directly in the code without any explicit explanation of its meaning or purpose. It's "magic" because its origin and significance are not immediately obvious to someone reading the code.==

**Key Characteristics of Magic Numbers:**

- **Unexplained Value:** The number's relevance is not clear from context alone.

- **Hardcoded:** It's directly typed into the code, not assigned to a named variable or constant.

- **Potential for Repetition:** The same magic number might appear in multiple, disparate parts of the codebase.


**How to Address Magic Numbers:**

The solution to magic numbers is to ==replace them with **named constants**==. This involves giving a meaningful name to the numeric literal, thus clearly communicating its purpose.

In C++, there are several ways to define named constants:

1. **`const` Keyword:** This is the most common and generally preferred way to define compile-time constants within a specific scope (e.g., local to a function, member of a class).

 ```c++
 const int MAX_USERS = 100;
 const double PI = 3.14159;
 ```

 `const` variables must be initialized at the time of declaration.

2. **[[C++ - constexpr]] Keyword (C++11 and later):** `constexpr` is an extension of `const`. It indicates that the value can be evaluated at compile-time. This is often used for constants that are part of expressions that need to be known at compile time (e.g., array sizes, template arguments).

 ```c++
 constexpr int BUFFER_SIZE = 1024;
 constexpr double GRAVITY_ACCELERATION = 9.81;
 ```

 Using `constexpr` where possible can enable more compile-time optimizations.

3. **`enum` (Enumerations):** For a set of related integer constants, `enum` is a good choice. `enum class` (scoped enums, C++11) is preferred over plain `enum` for better type safety and to avoid name collisions.

 ```c++
 enum class ErrorCode {
 SUCCESS = 0,
 FILE_NOT_FOUND = 1,
 PERMISSION_DENIED = 2
 };
 ```

4. **`

# define` (Preprocessor Macro):** While technically possible, `

# define` macros are generally discouraged for defining constants in modern C++ because they are handled by the preprocessor (before compilation), do not have type safety, and can lead to subtle bugs.

 ```c++

# define MAX_ATTEMPTS 3 // Discouraged for constants
 ```

 Using `const` or `constexpr` is always preferred over `

# define` for numeric constants.


**Benefits of using Named Constants:**

- **Self-documenting code:** The name itself explains the number's purpose.

- **Easier Maintenance:** If the value needs to change, you only modify it in one place (the constant definition). All uses of the constant will automatically reflect the change.

- **Reduced Errors:** No risk of typos when reusing the same conceptual value.

- **Improved Readability and Debugging:** Makes code easier to follow and reason about.

# **Related Concepts:**

---
- **Literals:** A **literal** is a fixed value in source code. `5`, `3.14`, `'A'`, `"hello"` are all literals. Magic numbers are a _type_ of literal that causes issues. The concept of addressing magic numbers is about replacing problematic literals with named constants.

- **Constants:** A **constant** is a value that cannot be changed after it is initialized. Named constants (like those defined with `const` or `constexpr`) are the direct solution to magic numbers. They provide a symbolic name for a literal, making its meaning clear and allowing for easy modification in a single location.

- **Enums (Enumerations):** As mentioned, enums are a way to define a set of named integer constants. They are particularly useful when a variable can take on one of a limited set of discrete values, providing type safety and readability over using raw integer literals.

- **Code Smell:** A **code smell** is a surface indication that there is a deeper problem in the system. Magic numbers are a classic example of a code smell because their presence often indicates a lack of clarity, poor design, or potential maintenance headaches. Addressing code smells improves code quality without necessarily changing the program's functionality.

- **DRY (Don't Repeat Yourself) Principle:** This software development principle aims to reduce repetition of information. Magic numbers often violate the DRY principle because the same logical value is repeated throughout the codebase. Replacing them with a single named constant adheres to DRY.

- **Readability and Maintainability:** These are fundamental qualities of good software engineering. The practice of eliminating magic numbers directly contributes to improving both readability (code is easier to understand) and maintainability (code is easier to change and fix).

# **Examples:**

---
```c++

# include <iostream>

# include <string>

# include <vector>

// Example demonstrating the problem with "Magic Numbers"
void processOrder_magic_numbers(int quantity, double price) {
 // What do 0 and 0 mean here? They are magic numbers.
 // Is 0 a tax rate? A discount?
 // Is 0 another tax? A different discount?
 double total = quantity * price;
 if (total > 100.0) { // What does 100 represent? A threshold for what?
 total = total - (total * 0.1); // Applying some calculation
 }
 total = total + (total * 0.05); // Applying another calculation
 std::cout << "Order total (with magic numbers): " << total << std::endl;
}

// Example demonstrating the solution: using named constants
// Option 1: Using 'const' for compile-time constants (most common)
const double DISCOUNT_RATE = 0.10; // Clearly indicates a 10% discount
const double SALES_TAX_RATE = 0.05; // Clearly indicates a 5% sales tax
const double DISCOUNT_THRESHOLD = 100.0; // Clearly indicates the total amount needed for a discount

void processOrder_named_constants(int quantity, double price) {
 double total = quantity * price;
 if (total > DISCOUNT_THRESHOLD) { // Readable: if total exceeds the discount threshold
 total = total - (total * DISCOUNT_RATE); // Readable: apply the discount rate
 }
 total = total + (total * SALES_TAX_RATE); // Readable: add the sales tax rate
 std::cout << "Order total (with named constants): " << total << std::endl;
}

// Example using 'constexpr' for compile-time evaluation (C++11+)
// 'constexpr' guarantees compile-time evaluation if possible, often for values
// that need to be known at compile time, like array sizes.
constexpr int MAX_BUFFER_SIZE = 256; // Defines a constant buffer size
constexpr double PI = 3.1415926535; // A mathematical constant

void constexprExample {
 std::cout << "\n
---
Constexpr Example
---
" << std::endl;
 char buffer[MAX_BUFFER_SIZE]; // Array size determined by constexpr
 std::cout << "Buffer size: " << sizeof(buffer) << " bytes" << std::endl;
 std::cout << "Value of PI: " << PI << std::endl;
 // (Actual use of buffer omitted for brevity)
}

// Example using 'enum class' for related integer constants (C++11+)
// 'enum class' provides type safety and avoids name collisions compared to plain 'enum'.
enum class LogLevel {
 INFO = 0,
 WARNING = 1,
 ERROR = 2,
 DEBUG = 3
};

void logMessage(LogLevel level, const std::string& message) {
 std::cout << "Log Level: ";
 switch (level) {
 case LogLevel::INFO: std::cout << "INFO"; break;
 case LogLevel::WARNING: std::cout << "WARNING"; break;
 case LogLevel::ERROR: std::cout << "ERROR"; break;
 case LogLevel::DEBUG: std::cout << "DEBUG"; break;
 default: std::cout << "UNKNOWN"; break;
 }
 std::cout << " - " << message << std::endl;
}

void enumClassExample {
 std::cout << "\n
---

Enum Class Example
---
" << std::endl;
 // Instead of: logMessage(0, "User logged in."); // Magic number 0
 logMessage(LogLevel::INFO, "User logged in.");
 logMessage(LogLevel::ERROR, "Failed to connect to database.");
 // No magic numbers, clear meaning.
}

int main {
 std::cout << "Comparing Magic Numbers vs. Named Constants:\n";

 // Scenario where magic numbers are problematic
 processOrder_magic_numbers(10, 5.0); // Total: 50. No discount applied.
 processOrder_magic_numbers(10, 12.0); // Total: 120. Discount applied.

 std::cout << "\nAfter refactoring with named constants:\n";
 // Now the logic is clear
 processOrder_named_constants(10, 5.0);
 processOrder_named_constants(10, 12.0);

 constexprExample;
 enumClassExample;

 return 0;
}
```

# **Flashcards:**

---
What is a "magic number" in programming?;;A numeric literal (a directly typed number) in code whose meaning is not immediately clear.

Why are magic numbers considered a "code smell"?;;They harm readability, make code difficult to maintain (hard to change multiple instances), and can lead to errors.

How should magic numbers be replaced in C++?;;By using named constants, typically defined with const or constexpr, or by using enum for related integer constants.