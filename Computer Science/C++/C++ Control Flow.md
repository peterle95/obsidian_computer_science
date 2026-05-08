---
memory: to_finish
tags:
 - learned
language:
 - C++
review-date: ""
last-reviewed: 2025-07-16
scheda: done
visit-count: 1
confidence-level: 1
consecutive-correct: 1
last-struggle-date: ""
cssclasses:

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
By default, a C++ program executes statements sequentially. However, to create programs that can make decisions, repeat actions, and respond to different inputs, we need to alter this linear flow. C++ control flow statements provide the essential mechanism to direct the order of execution based on certain conditions or for a specific number of iterations. This allows for the creation of dynamic, intelligent, and efficient programs that can handle complex tasks. Without control flow, programs would be simple, single-path scripts, incapable of handling the variability of real-world problems.

# **Core Explanation:**

---

C++ control flow refers to the order in which the program's statements are executed. It is managed through various statements that allow the program to branch, iterate, and jump to different sections of the code. These statements can be broadly categorized as:

- **Selection Statements:** These statements allow the program to choose which block of code to execute based on a condition.

 - `if`: Executes a block of code if a specified condition is true.

 - `if-else`: Executes one block of code if a condition is true and another block if it is false.

 - `if-else if-else`: Allows for checking multiple conditions in sequence.

 - `switch`: A multi-way branch statement that compares the value of a variable to a list of case values.

- **Iteration Statements (Loops):** These statements are used to execute a block of code repeatedly as long as a certain condition is met.

 - `for`: Executes a block of code a specific number of times. It's ideal when the number of iterations is known beforehand.

 - `while`: Executes a block of code as long as a specified condition is true. The condition is checked _before_ each iteration.

 - `do-while`: Similar to a `while` loop, but the condition is checked _after_ each iteration, guaranteeing that the block of code is executed at least once.

- **Jump Statements:** These statements allow for the unconditional transfer of control to another part of the program.

 - `break`: Terminates the enclosing loop or `switch` statement.

 - `continue`: Skips the current iteration of a loop and proceeds to the next one.

 - `goto`: Transfers control to a labeled statement within the same function. Its use is generally discouraged as it can lead to unstructured and difficult-to-read code.

 - `return`: Exits the current function and returns a value to the caller.

# **Related Concepts:**

---
- **Boolean Logic:** Control flow statements heavily rely on boolean expressions (`true` or `false`) to make decisions. The conditions in `if`, `while`, and `for` statements are all boolean expressions.

- **Functions:** Functions are blocks of code that perform a specific task and can be called from other parts of the program. Control flow dictates the execution within a function, and the `return` statement is a jump statement that transfers control back from a function.

- **Algorithms:** An algorithm is a step-by-step procedure for solving a problem. Control flow statements are the building blocks used to implement the logic of an algorithm in code.

- **Recursion:** A function that calls itself is recursive. This is an alternative to iteration for performing repetitive tasks and represents a different kind of control flow.

# **Examples:**

---
#

#

# Selection Statements
```c++

# include <iostream>

int main {
 //
---
if-else if-else statement
---
int score = 85;
 std::cout << "
---
if-else if-else Example
---
" << std::endl;
 // Checks the value of 'score' and prints the corresponding grade.
 if (score >= 90) {
 std::cout << "Grade: A" << std::endl;
 } else if (score >= 80) {
 std::cout << "Grade: B" << std::endl; // This block will be executed
 } else if (score >= 70) {
 std::cout << "Grade: C" << std::endl;
 } else {
 std::cout << "Grade: F" << std::endl;
 }

 //
---
switch statement
---
char grade = 'B';
 std::cout << "\n
---
switch Example
---
" << std::endl;
 // Evaluates the 'grade' variable and executes the corresponding case.
 switch (grade) {
 case 'A':
 std::cout << "Excellent!" << std::endl;
 break; // Exits the switch statement
 case 'B':
 std::cout << "Good job!" << std::endl; // This block will be executed
 break;
 case 'C':
 std::cout << "You passed." << std::endl;
 break;
 default:
 std::cout << "Invalid grade." << std::endl;
 }
 return 0;
}
```

#

#

# Iteration Statements
```c++

# include <iostream>

int main {
 //
---
for loop
---
std::cout << "
---
for loop Example
---
" << std::endl;
 // Prints numbers from 1 to 5.
 // The loop initializes i to 1, continues as long as i is less than or equal to 5, and increments i after each iteration.
 for (int i = 1; i <= 5; ++i) {
 std::cout << i << " ";
 }
 std::cout << std::endl;

 //
---
while loop
---
std::cout << "\n
---
while loop Example
---
" << std::endl;
 int n = 5;
 // Prints numbers from 5 down to 1.
 // The loop continues as long as n is greater than 0.
 while (n > 0) {
 std::cout << n << " ";
 n--; // Decrement n in each iteration
 }
 std::cout << std::endl;

 //
---
do-while loop
---
std::cout << "\n
---
do-while loop Example
---
" << std::endl;
 int count = 1;
 // Prints numbers from 1 to 3.
 // The code block is executed once before the condition is checked.
 do {
 std::cout << "Count: " << count << std::endl;
 count++;
 } while (count <= 3);

 return 0;
}
```

#

#

# Jump Statements
```c++

# include <iostream>

int main {
 //
---
break and continue
---
std::cout << "
---
break and continue Example
---
" << std::endl;
 for (int i = 1; i <= 10; ++i) {
 if (i == 3) {
 continue; // Skips the rest of the loop body for i = 3 and proceeds to the next iteration.
 }
 if (i == 8) {
 break; // Terminates the loop when i becomes 8.
 }
 std::cout << i << " "; // This line will not be executed for i = 3.
 }
 std::cout << std::endl;

 return 0;
}
```

# **Flashcards:**

---
What are the three main categories of control flow statements in C++?;;Selection (if, switch), Iteration (for, while, do-while), and Jump (break, continue, goto, return).

What is the primary difference between a while loop and a do-while loop in C++?;;A while loop checks its condition before each iteration, so it may not execute at all. A do-while loop checks its condition after each iteration, guaranteeing the loop body executes at least once.

What is the purpose of the break and continue statements within a loop?;;break immediately terminates the entire loop. continue skips the current iteration of the loop and proceeds to the next one.