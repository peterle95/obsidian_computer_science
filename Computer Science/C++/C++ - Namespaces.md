---
memory: to_finish
tags:
 - learned
language:
 - C++
review-date: ""
last-reviewed: 2025-08-10
scheda: done
visit-count: 4
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-07-15

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

A **namespace** in C++ is a mechanism for grouping related declarations (like classes, functions, variables, and types) into a named scope. This organization helps prevent **name clashes**, especially in large projects where multiple libraries or modules might use the same identifier for different purposes. Imagine namespaces as separate "folders" for your code, preventing accidental conflicts when two files have a function or variable with the same name.

#

#

# Using the `std` Namespace

The C++ [[Standard Library]] is defined within the `std` namespace. So, when you use components like `cout`, `cin`, `string`, or `vector`, you're actually referring to `std::cout`, `std::cin`, and so on. To simplify the code and avoid repeated qualification with `std::`, you can use the following:

```cpp
using namespace std;

cout << "Hello, world!" << endl; // Instead of std::cout
```

However, it's important to note that using `using namespace std;` in your code can lead to potential name conflicts if you're also using other namespaces. In that case, you may want to use a more targeted approach:

```cpp

# include <iostream>

# include <string>

int main {
 std::string my_string = "Hello, world!";
 std::cout << my_string << std::endl; // Use std:: for specific names
 return 0;
}
```

#

#

# Defining a Namespace

You can create your own namespace using the `namespace` keyword followed by the namespace name and curly braces to enclose its contents:

```cpp
namespace MyGraphicsLibrary {
 class Point { /* ... */ };
 class Line { /* ... */ };
 // ... other graphics-related declarations
}
```

#

#

# Accessing Members

To use a member defined within a namespace, you qualify the name with the namespace and the scope resolution operator (`::`):

```cpp
MyGraphicsLibrary::Point p1; // Create a Point object from MyGraphicsLibrary
```

#

#

# Using Declarations

Repeatedly writing the fully qualified name can be tedious. A **using declaration** allows you to bring a specific name from a namespace into your current scope:

```cpp
using MyGraphicsLibrary::Point; // Now you can just write "Point" instead of "MyGraphicsLibrary::Point"
Point p2; // Refers to MyGraphicsLibrary::Point
```

#

#

# Using Directives

**1. `using namespace std;` (Using Directive)**

- **What it does:** This statement **brings all the names (identifiers)** from the `std` namespace into the current scope (wherever you place this `using` directive). This means you can use `cout`, `cin`, `string`, `vector`, `endl`, etc., directly without prefixing them with `std::`.
- **Analogy:** Imagine a huge library with many, many books. `using namespace std;` is like saying, "I'm going to take _all_ the books from the 'Standard Library' section and put them directly on my desk so I can use them easily without constantly walking to that section."
- **Pros:**
 - **Convenience:** It makes your code more concise, as you don't have to type `std::` repeatedly.
- **Cons (and why it's generally discouraged in larger projects and header files):**
 - **Name Collisions (Ambiguity):** The `std` namespace is vast. If you declare your own function or variable with the same name as something in `std` (e.g., `count`, `list`, `vector`), the compiler won't know which one you mean, leading to compilation errors or unexpected behavior. This problem becomes much more likely in larger projects where you might be combining code from different sources or using third-party libraries.
 - **Reduced Readability/Clarity:** It can make it harder to tell at a glance where a particular function or type is coming from, especially for someone else reading your code.
 - **Polluting the Global Namespace:** When used at the global scope (outside of any function), it effectively "dumps" all `std` names into your global namespace, making it more crowded and increasing the risk of conflicts.
 - **Avoid in Header Files:** This is a crucial rule. Never put `using namespace std;` in a header file (`.h` or `.hpp`). If you do, any `.cpp` file that includes that header will also inherit the `using namespace std;` directive, potentially causing name conflicts in unrelated parts of your project.

**2. `using std::string, std::cout, std::endl;` (Using Declarations)**

- **What it does:** This statement **selectively brings specific names** from the `std` namespace into the current scope. You explicitly list each identifier you want to use without the `std::` prefix.
- **Analogy:** Using the library analogy, this is like saying, "I'm only going to take _these specific books_ (e.g., 'Cout', 'String', 'Endl') from the 'Standard Library' section and put them on my desk."
- **Pros:**
 - **Avoids Name Collisions:** By only importing the names you explicitly need, you significantly reduce the chance of conflicts with your own code or other libraries.
 - **Improved Readability/Clarity:** It's clear exactly which `std` elements you are choosing to use unqualified.
 - **Minimal Pollution:** You only bring what's necessary into the current scope.
- **Cons:**
 - **More Typing:** You have to list each element you want to use without the `std::` prefix. For a very small program that uses many `std` elements, this might seem a bit cumbersome.

**When to use which:**

- **For small, self-contained examples or quick tests:** `using namespace std;` might be acceptable for convenience, especially if you're certain there won't be any name conflicts.
- **For most real-world, larger C++ projects (and always in header files):** **It is highly recommended to either:**
 - **Fully qualify names:** `std::cout << "Hello";` `std::string myString;`
 - **Use `using` declarations for specific elements:** `using std::cout; using std::endl;` for the few items you use very frequently, and then fully qualify others.
 - **Limit the scope of `using namespace std;`:** If you absolutely must use it, consider putting `using namespace std;` inside a function or a specific block of code, rather than at the global scope, to limit its impact.

In summary, the key difference is the **scope and specificity** of what you bring into your current naming environment. `using namespace std;` is broad and can lead to problems, while `using std::string, std::cout, std::endl;` is precise and generally considered better practice for maintainable and robust code.