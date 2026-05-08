---
memory: to_finish
tags:
 - learned
language:
 - C++
review-date: ""
last-reviewed: 2025-08-13
scheda: done
visit-count: 3
confidence-level: 2
consecutive-correct: 3
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
Templates solve the **code duplication problem** that occurs when you need the same algorithm to work with multiple data types. Instead of writing identical logic multiple times for different types (function overloading), templates allow you to write generic code once that automatically adapts to any compatible type.

This is crucial in C++ because:
- **Type Safety**: C++ is statically typed, so you can't just use `void*` like in C
- **Performance**: No runtime overhead - templates generate optimized code for each type
- **Maintainability**: One implementation to debug, test, and maintain
- **Extensibility**: Works with future types without modifying existing code
- **Standard Library Foundation**: STL containers, algorithms, and utilities are built on templates

# **Core Explanation:**

---
**Templates** are a ==compile-time code generation mechanism that creates type-specific functions/classes from generic blueprints. When you call a template function, the compiler performs **template instantiation** - it generates a concrete function by substituting the template parameters with actual types==.

**Key Characteristics:**
- **Compile-time**: All template processing happens during compilation
- **Type Deduction**: Compiler automatically determines template parameters from arguments
- **Zero Runtime Overhead**: Generated code is as efficient as hand-written type-specific code
- **Lazy Instantiation**: Only used template specializations are compiled
- **Type Requirements**: Template code must be valid for all types it's instantiated with

**Function Overloading** requires manually writing separate functions for each type, leading to:
- Code duplication (same logic, different types)
- Maintenance burden (fixing bugs in multiple places)
- Inability to work with unknown future types
- Larger binary size (all overloads compiled regardless of usage)

**Template Advantage**: Write once, use everywhere - the compiler becomes your code generator.

# **Related Concepts:**

---
**Template Specialization**: Providing specific implementations for particular types when the generic version isn't suitable. Complements the generic approach.

**SFINAE (Substitution Failure Is Not An Error)**: Mechanism that allows templates to gracefully handle type incompatibilities during compilation.

**Concepts (C++20)**: Constraints that specify requirements for template parameters, making templates more readable and providing better error messages.

**Generic Programming**: Programming paradigm where templates are the primary tool - writing code that works with any type meeting certain requirements.

**Metaprogramming**: Using templates to perform computations at compile-time, treating types and values as data.

**Type Traits**: Template-based utilities that provide information about types at compile-time, enabling conditional compilation.

**Overload Resolution**: The process by which the compiler chooses between multiple function candidates (overloads, templates, specializations).

# **Examples:**

---
```cpp
// FUNCTION OVERLOADING APPROACH (Manual, Repetitive)
// You must write separate functions for each type you want to support

void swap_int(int& a, int& b) {
 int temp = a; // Same logic
 a = b; // for every
 b = temp; // single type
}

void swap_double(double& a, double& b) {
 double temp = a; // Duplicate code
 a = b; // maintenance nightmare
 b = temp; // what if we find a bug?
}

void swap_string(std::string& a, std::string& b) {
 std::string temp = a; // Must fix in ALL functions
 a = b;
 b = temp;
}
// Need to add more functions for every new type!

// TEMPLATE APPROACH (Automatic, Scalable)
// One template generates all the functions you need

template<typename T> // T is a placeholder for any type
void swap(T& a, T& b) {
 T temp = a; // This code works for ANY type T
 a = b; // that supports copy constructor
 b = temp; // and assignment operator
}

// Usage examples showing automatic type deduction
int main {
 // Template automatically creates swap<int> version
 int x = 5, y = 10;
 swap(x, y); // Compiler generates: swap<int>(int&, int&)

 // Template automatically creates swap<std::string> version
 std::string s1 = "hello", s2 = "world";
 swap(s1, s2); // Compiler generates: swap<std::string>(std::string&, std::string&)

 // Template works with custom types too!
 class Point {
 int x, y;
 public:
 Point(int a, int b) : x(a), y(b) {}
 // As long as Point has copy constructor and assignment operator,
 // the template will work automatically
 };

 Point p1(1, 2), p2(3, 4);
 swap(p1, p2); // Compiler generates: swap<Point>(Point&, Point&)

 // Even complex types work without modification!
 std::vector<int> vec1 = {1, 2, 3}, vec2 = {4, 5, 6};
 swap(vec1, vec2); // Compiler generates: swap<std::vector<int>>(...)

 return 0;
}

// COMPARISON: Template vs Overloading for min/max functions
// With overloading, you'd need:
int min_int(int a, int b) { return (a < b) ? a : b; }
double min_double(double a, double b) { return (a < b) ? a : b; }
std::string min_string(const std::string& a, const std::string& b) {
 return (a < b) ? a : b;
}
// ... and so on for every type

// With templates, you need only:
template<typename T>
const T& min(const T& a, const T& b) {
 return (a < b) ? a : b; // Works with any type that has operator<
}

// ADVANCED EXAMPLE: Template constraints
template<typename T>
void print_if_numeric(T value) {
 // This template only works with numeric types
 // If T doesn't support arithmetic, compilation fails with clear error
 std::cout << "Value: " << value << ", doubled: " << value * 2 << std::endl;
}

// The template approach scales infinitely without code changes
// The overloading approach requires manual work for each new type
````

# **Flashcards:**

---
What is the main problem that templates solve compared to function overloading?;; Templates eliminate code duplication by allowing you to write generic code once that works with multiple types, instead of manually writing separate functions for each type.

When does template instantiation occur and what does it create?;; Template instantiation occurs at compile-time when the compiler generates concrete, type-specific functions from the generic template blueprint based on the actual types used in function calls.

What are the performance implications of using templates vs function overloading?;; Templates have zero runtime overhead because they generate optimized type-specific code at compile-time, and only instantiate versions that are actually used, while function overloading compiles all versions regardless of usage.

What requirements must a type T satisfy to work with a template function?;; The type T must support all operations used within the template (e.g., copy constructor, assignment operator, comparison operators), and the template code must be syntactically valid when T is substituted.

How do templates handle unknown or future types compared to function overloading?;; Templates automatically work with any compatible type (including future custom types) without modifying the original code, while function overloading requires manually writing new overloaded functions for each new type.

What is the key difference in maintenance between templates and function overloading?;; With templates, bug fixes and improvements are made in one place and automatically apply to all type instantiations, while function overloading requires fixing the same bug in multiple separate functions.