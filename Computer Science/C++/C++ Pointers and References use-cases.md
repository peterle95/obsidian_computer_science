---
memory: to_finish
tags:
 - learned
language:
 - C++
review-date: ""
last-reviewed: 2025-08-06
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-07-06

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

On the surface, both references and pointers are very similar as both are used to have one variable provide access to another. With both providing lots of the same capabilities, it's often unclear what is different between these mechanisms. In this article, I will try to illustrate the differences between pointers and references.

[[Distinguishing Between Pointers, Dereferenced Pointers, and References]]

#

# Pointers
<mark style="background:

# FFB86CA6;">A pointer is a variable that holds the memory address of another variable</mark>. ==A pointer needs to be dereferenced with the `*` operator to access the memory location it points to.==

> "A pointer is an object that stores the memory address of another object."

#

# References
A reference variable is an alias, that is, another name for an already existing variable. A<mark style="background:

# D2B3FFA6;"> reference, like a pointer, is also implemented by storing the address of an object</mark>. A reference <mark style="background:

# FFB8EBA6;">can be thought of as a constant pointer</mark> (not to be confused with a pointer to a constant value!) with automatic indirection, i.e., the compiler will apply the `*` operator for you.

> "A reference acts as an alternative name for an object. Once initialized, a reference cannot be made to refer to a different object. You can think of it as an immutable pointer that is automatically dereferenced."
>
> "To declare a reference, you use the & symbol in the type, not in the initializer."
>
> "When you assign to a reference, you change the value of the object referred to, not the reference itself."
>
> "Assigning references performs a deep copy (assigns to the referred-to object), whereas assigning pointers does not (assigns to the pointer object itself)."

Understanding the differences between pointers and references is crucial for effective C++ programming. The **C++ Course** explains these concepts in depth, helping you choose the right approach in your code.

```cpp
int i = 3;

// A pointer to variable i or "stores the address of i"
int *ptr = &i;

// A reference (or alias) for i.
int &ref = i;
```

#

# Pointers and References

References are aliases to existing variables, implemented by storing object addresses with automatic dereferencing.

**Key difference in assignment operations:**
- When assigning to a reference (`ref = 10`), you modify the original object's value
- When assigning to a pointer (`ptr = &anotherVar`), you change what the pointer points to

```cpp
// Reference example
int original = 5;
int& ref = original;
ref = 10; // original is now 10

// Pointer example
int original = 5;
int* ptr = &original;
ptr = &anotherVar; // ptr now points elsewhere, original unchanged
```

#

# Differences

1. **Initialization**:
 - Pointers can be declared and initialized in one step or in multiple lines.
 - References must be declared and initialized in a single step.

2. **Reassignment**:
 - Pointers can be reassigned to point to different variables.
 - References cannot be reassigned, they must be assigned at initialization.

3. **Memory Address**:
 - Pointers have their own memory address and size on the stack.
 - References share the same memory address with the original variable and take up no space on the stack.

4. **NULL value**:
 - Pointers can be assigned `NULL` directly.
 - References cannot be assigned `NULL`.

5. **Indirection**:
 - Pointers can have pointers to pointers (double pointers).
 - References only offer one level of indirection.

6. **Arithmetic operations**:
 - Pointers support various arithmetic operations.
 - There is no such thing as Reference Arithmetic.

For a detailed comparison, please refer to the table below:

| Feature | References | Pointers |
|
---

---

---

---
-- |
---

---

---

---

---

---

---

---

---

---

---

---

---

---

---

---
|
---

---

---

---

---

---

---

---

---

---

---
- |
| Reassignment | Cannot be reassigned | Can be reassigned |
| Memory Address | Shares the same address as the original variable | Have their own memory address |
| Work | Refers to another variable | Stores the address of the variable |
| Null Value | Cannot have a null value | Can have a null value assigned |
| Arguments | Passed by value | Passed by reference |

# When to Use Pointers vs References

The performances are exactly the same as references are implemented internally as pointers. But still, you can keep some points in your mind to decide when to use what:

#

# Use References
* In function parameters and return types.
* "When you want to modify an object passed to a function: Passing by reference avoids making a copy of the object, which can be more efficient for large objects."
* "When "no object" (represented by nullptr) is not a valid argument: Using a reference signals that a valid object must be provided."

#

# Use Pointers
* If pointer arithmetic or passing a `NULL` pointer is needed. For example, for arrays (Note that accessing an array is implemented using pointer arithmetic).
* To implement data structures like a linked list, a tree, etc. and their algorithms. This is so because, in order to point to different cells, we have to use the concept of pointers.
* "When "no object" (represented by nullptr) is a valid argument: Pointers can be checked for nullptr to handle cases where an object might not be available."
* "When you need to point to different objects at different times in a program: Unlike references, pointers can be reassigned to point to different objects."

> "Use references when you can, and pointers when you have to. References are usually preferred over pointers whenever you don't need "reseating". This usually means that references are most useful in a class's public interface. References typically appear on the skin of an object, and pointers on the inside."

The exception to the above is where a function's parameter or return value needs a "sentinel" reference — a reference that does not refer to an object. This is usually best done by returning/taking a pointer, and giving the "`nullptr`" value this special significance (references must always alias objects, not a dereferenced null pointer).

# Flashcards

---
What is a pointer?;; A pointer is a variable that holds the memory address of another variable. It needs to be dereferenced with the * operator to access the memory location it points to.

What is a reference?;; A reference variable is an alias, another name for an already existing variable, implemented by storing the address of an object with automatic dereferencing.

What is a key difference in assignment operations between pointers and references?;; When assigning to a reference (ref = 10), you modify the original object's value. When assigning to a pointer (ptr = &anotherVar), you change what the pointer points to.

Can pointers and references be reassigned?;; Pointers can be reassigned to point to different variables, but references cannot be reassigned; they must be assigned at initialization.

Do pointers and references have their own memory addresses?;; Pointers have their own memory address and size on the stack. References share the same memory address with the original variable and take up no space on the stack.

When should you generally prefer using references over pointers?;; Use references when you don't need "reseating" (reassigning to a different object), especially in function parameters and return types where you want to modify an object passed to a function or when "no object" (nullptr) is not a valid argument.