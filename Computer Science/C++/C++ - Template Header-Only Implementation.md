---
memory: to_finish
tags:
 - learned
language:
 - C++
review-date: ""
last-reviewed: 2025-08-14
scheda: done
visit-count: 3
confidence-level: 1
consecutive-correct: 1
last-struggle-date: 2025-08-02
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
Template Header-Only Implementation solves the fundamental problem of **template compilation and linking**. Unlike regular C++ functions that can be declared in headers and defined in source files, ==templates must have their complete definition available at compile time in every translation unit that uses them.==

This is critical because:

- **Templates are not actual code** - <mark style="background:

# FF5582A6;">they're blueprints that the compiler uses to generate real code</mark>
- **Instantiation happens at compile time** - the compiler needs to see the full template definition to create specific versions (like `vector<int>` from `vector<T>`)
- **Each translation unit must be able to instantiate independently** - without this, linking would fail with "undefined reference" errors

This concept is essential for understanding why template libraries like STL have everything in headers, and why you can't separate template declarations from definitions like you can with regular functions.

# **Core Explanation:**

---
**Template Header-Only Implementation** means that template definitions (not just declarations) must be placed in header files and included wherever the template is used.

**Key Characteristics:**

- **Complete definition required**: <mark style="background:

# FF5582A6;">Templates need their full implementation visible at compile time</mark>
- **No separate compilation**: Unlike regular functions, you can't compile template definitions separately
- **Instantiation on demand**: The compiler generates specific versions only when needed
- **Multiple inclusion safe**: Templates can be included in multiple translation units without violating ODR (One Definition Rule)

**How it works:**

1. **Compilation phase**: When the compiler encounters a template use (like `Array<int>`), it looks for the complete template definition
2. **Instantiation**: The compiler generates the specific code for that type combination
3. **Linking phase**: The linker discards duplicate instantiations, keeping only one copy

**Why regular functions are different:**

- Regular functions can be declared in headers and defined in source files because they exist as actual compiled code
- Templates exist only as patterns until instantiated, so the pattern must be available everywhere it's used

# **Related Concepts:**

---
**Template Instantiation** - The process by which templates become actual code; requires the full template definition to be available.

**Translation Units** - Each .cpp file and its included headers form a translation unit; templates must be fully defined in each unit that uses them.

**One Definition Rule (ODR)** - C++ rule that normally prevents multiple definitions; templates are exempt because the compiler can discard duplicate instantiations.

**Template Compilation Model** - The overall process of how templates are compiled, of which header-only implementation is a key requirement.

**Separate Compilation** - Traditional C++ model where declarations go in headers and definitions in source files; doesn't work for templates.

**Include Guards/Pragma Once** - Still necessary for template headers to prevent multiple inclusions within the same translation unit.

**Template Specialization** - Also follows header-only rule; specialized versions must be in headers.

# **Examples:**

---
```cpp
// ❌ WRONG WAY - This will NOT work for templates
// MyTemplate.h
template<typename T>
class MyTemplate {
public:
 void doSomething(T value); // Only declaration in header
};

// MyTemplate.cpp - Template definition in source file
template<typename T>
void MyTemplate<T>::doSomething(T value) {
 // Implementation here
}
// ❌ This causes linking errors because the compiler can't find
// the template definition when it needs to instantiate MyTemplate<int>
```

```cpp
// ✅ CORRECT WAY - Header-only implementation
// MyTemplate.h

# ifndef MYTEMPLATE_H

# define MYTEMPLATE_H

template<typename T>
class MyTemplate {
private:
 T* data;
 size_t size;

public:
 // Constructor definition directly in header
 MyTemplate(size_t n) : size(n) {
 data = new T[n]; // Compiler can see this when instantiating
 }

 // Method definition in header
 void doSomething(T value) {
 // When user writes MyTemplate<int> obj(5);
 // compiler generates: void doSomething(int value) { ... }
 for (size_t i = 0; i < size; ++i) {
 data[i] = value;
 }
 }

 // Destructor also in header
 ~MyTemplate {
 delete data;
 }
};

# endif
// ✅ Complete template definition is available wherever this header is included
```

```cpp
// Alternative: Definition after declaration in same header
// Array.hpp

# ifndef ARRAY_HPP

# define ARRAY_HPP

template<typename T>
class Array {
private:
 T* elements;
 size_t arraySize;

public:
 // Declarations only
 Array;
 Array(unsigned int n);
 ~Array;
 T& operator;
 size_t size const;
};

// Definitions in the same header file (after class declaration)
template<typename T>
Array<T>::Array : elements(nullptr), arraySize(0) {
 // Default constructor implementation
 // Must be in header so compiler can instantiate Array<int>, Array<string>, etc.
}

template<typename T>
Array<T>::Array(unsigned int n) : arraySize(n) {
 // Parameterized constructor
 // When user writes Array<double> arr(10);
 // compiler generates this constructor specifically for double
 elements = new T[n];
}

template<typename T>
Array<T>::~Array {
 // Destructor implementation
 // Compiler needs this to generate proper cleanup code for each type
 delete elements;
}

template<typename T>
T& Array<T>::operator {
 // Operator overloading implementation
 // For Array<string> arr; arr = "hello";
 // compiler generates: string& operator
 return elements[index];
}

template<typename T>
size_t Array<T>::size const {
 // Const member function
 // Available for instantiation with any type
 return arraySize;
}

# endif
// ✅ All template definitions are in the header file
```

```cpp
// Usage example - main.cpp

# include "Array.hpp" // Includes complete template definitions

int main {
 // When compiler sees this line, it looks for Array<int> constructor
 // It finds the template definition in the header and instantiates:
 // Array<int>::Array(unsigned int n) { elements = new int[n]; }
 Array<int> intArray(5);

 // Compiler instantiates: int& Array<int>::operator
 intArray = 42;

 // Compiler instantiates: Array<string>::Array(unsigned int n)
 Array<std::string> stringArray(3);

 // Compiler instantiates: string& Array<string>::operator
 stringArray = "Hello";

 return 0;
}
// ✅ All instantiations successful because template definitions were available
```

```cpp
// Function template example
// utils.hpp

# ifndef UTILS_HPP

# define UTILS_HPP

// Function template declaration AND definition in header
template<typename T>
void swap(T& a, T& b) {
 // Implementation must be here, not in a .cpp file
 // When user calls swap(x, y) with integers:
 // compiler generates: void swap(int& a, int& b) { ... }
 T temp = a;
 a = b;
 b = temp;
}

template<typename T>
T max(const T& a, const T& b) {
 // Another function template - complete definition in header
 // For max(5, 10): compiler generates max<int>(const int&, const int&)
 // For max(3.14, 2.71): compiler generates max<double>(const double&, const double&)
 return (a > b) ? a : b;
}

# endif
// ✅ Function templates also follow header-only rule
```

# **Flashcards:**

---
Why must template definitions be in header files?;; Because templates are blueprints that need to be available at compile time for instantiation - the compiler must see the complete definition to generate specific versions for each type used.

What happens if you put a template definition in a .cpp file?;; You get linking errors because the compiler can't find the template definition when it needs to instantiate it in other translation units that include only the header.

How do templates differ from regular functions regarding compilation?;; Regular functions can be declared in headers and defined in source files because they exist as compiled code, while templates exist only as patterns until instantiated and need the full definition available everywhere they're used.

What is template instantiation and why does it require header-only implementation?;; Template instantiation is when the compiler generates actual code from a template blueprint for specific types - it requires the complete template definition to be visible at compile time.

Does header-only implementation violate the One Definition Rule (ODR)?;; No, templates are exempt from ODR because the compiler can discard duplicate instantiations during linking, keeping only one copy of each instantiated version.

What are the two ways to organize template code in headers?;; 1) Define everything inline within the class declaration, or 2) Declare the template class first, then provide all method definitions below in the same header file.