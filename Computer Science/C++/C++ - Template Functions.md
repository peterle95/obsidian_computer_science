---
memory: to_finish
tags:
 - learned
language:
 - C++
review-date: ""
last-reviewed: 2025-08-17
scheda: done
visit-count: 4
confidence-level: 2
consecutive-correct: 2
last-struggle-date: 2025-07-21
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
Template functions solve the problem of **code duplication** and enable **generic programming** in C++. Without templates, you would need to write separate functions for each data type (int, float, string, etc.), leading to repetitive code that's hard to maintain.

The fundamental problems templates solve:

- **Type-specific code duplication**: ==Instead of writing `swapInt`, `swapFloat`, `swapString`, you write one `swap<T>` template==
- **Compile-time type safety**: Templates are <mark style="background:

# FF5582A6;">resolved at compile time</mark>, ensuring type safety without runtime overhead
- **Performance**: <mark style="background:

# BBFABBA6;">No runtime polymorphism overhead like virtual functions</mark> - templates generate optimized code for each type used
- **Code maintainability**: One template definition means one place to fix bugs or add features

Templates are crucial in C++ because they enable the Standard Template Library (STL), generic algorithms, and modern C++ design patterns while maintaining C++'s zero-overhead principle.

# **Core Explanation:**

---
**Function templates** are blueprints for creating ==functions that can work with different data types==. They use **template parameters** to represent generic types that are specified when the function is called.

**Key characteristics:**

- **Generic**: Work with any type that satisfies the template's requirements
- **Compile-time**: Template instantiation happens during compilation, not runtime
- **Type deduction**: The compiler can often deduce template parameters from function arguments
- **Specialization**: You can provide specific implementations for certain types
- **Header-only**: Templates must be defined in header files (template definitions need to be visible at compile time)

**How it works:**

1. **Template declaration**: `template<typename T>` declares a template parameter `T`
2. **Template definition**: Function body uses `T` as a placeholder for the actual type
3. **Instantiation**: When called with specific types, the compiler generates actual function code
4. **Type checking**: Template requirements are checked at compile time

**Syntax:**

```cpp
template<typename T> // Template parameter declaration
ReturnType functionName(T param1, T param2) {
 // Function body using T
}
```

# **Related Concepts:**

---
**Class Templates**: Similar to function templates but for creating generic classes like std::vector<\T>, std::array<\T>. Function templates create generic functions, class templates create generic classes.

**Template Specialization**: Allows providing specific implementations for certain types. Connects to function templates by allowing customization of generic behavior.

**SFINAE (Substitution Failure Is Not An Error)**: Advanced template technique that allows templates to work only with types that meet certain requirements. Extends function template capabilities.

**[[C++ - Template Metaprogramming]]**: Using templates to perform computations at compile time. Function templates are building blocks for more complex metaprogramming techniques.

**Function Overloading**: Different from templates - overloading creates multiple functions with same name but different parameters, while templates create one generic function.

**Polymorphism**: Templates provide "compile-time polymorphism" (static) vs. virtual functions which provide "runtime polymorphism" (dynamic). Templates are faster but less flexible.

**Generic Programming**: Programming paradigm enabled by templates, focusing on writing code that works with multiple types while maintaining type safety.

# **Examples:**

---
```cpp

# include <iostream>

# include <string>

// Template function declaration and definition
// template<typename T> declares T as a template parameter
// T can be any type that supports the operations used in the function
template<typename T>
void swap(T& a, T& b) {
 // Create temporary copy of first parameter
 T temp = a; // T must have copy constructor
 a = b; // T must have assignment operator
 b = temp; // T must have assignment operator
 // No return value - function modifies parameters by reference
}

// Template function that returns a value
// const T& means we return a reference to avoid copying
template<typename T>
const T& min(const T& a, const T& b) {
 // T must support < operator for comparison
 // If a < b is true, return a, otherwise return b
 // When equal, returns second parameter (b) as specified
 return (a < b) ? a : b;
}

// Another template function demonstrating same concepts
template<typename T>
const T& max(const T& a, const T& b) {
 // T must support > operator for comparison
 // If a > b is true, return a, otherwise return b
 // When equal, returns second parameter (b) as specified
 return (a > b) ? a : b;
}

int main {
 // Example 1: Template instantiation with int type
 int a = 2;
 int b = 3;

 // :: prefix ensures we call our template functions, not std:: versions
 // Compiler deduces T = int from arguments
 ::swap(a, b); // Instantiates swap<int>
 std::cout << "a = " << a << ", b = " << b << std::endl;

 // Compiler instantiates min<int> and max<int>
 std::cout << "min( a, b ) = " << ::min(a, b) << std::endl;
 std::cout << "max( a, b ) = " << ::max(a, b) << std::endl;

 // Example 2: Template instantiation with string type
 std::string c = "chaine1";
 std::string d = "chaine2";

 // Same template functions, but now T = std::string
 // Compiler generates separate functions for string type
 ::swap(c, d); // Instantiates swap<std::string>
 std::cout << "c = " << c << ", d = " << d << std::endl;

 // String comparison uses lexicographic ordering
 std::cout << "min( c, d ) = " << ::min(c, d) << std::endl;
 std::cout << "max( c, d ) = " << ::max(c, d) << std::endl;

 // Example 3: Explicit template parameter specification
 // Sometimes you need to specify the type explicitly
 float x = 3.14f;
 double y = 2.71;
 // This would cause compilation error due to type mismatch:
 // ::swap(x, y); // Error: T cannot be both float and double

 // Solution: explicit casting or separate variables of same type
 float y_float = static_cast<float>(y);
 ::swap(x, y_float); // Now both are float

 return 0;
}

// Advanced example: Template function with multiple template parameters
template<typename T, typename U>
void printTypes(const T& first, const U& second) {
 // This template can work with two different types
 std::cout << "First: " << first << ", Second: " << second << std::endl;
}

// Template function with template parameter constraints (C++20)
// template<typename T>
// requires std::is_arithmetic_v<T> // Only works with numeric types
// T multiply(T a, T b) {
// return a * b;
// }
```

# **Flashcards:**

---
What is the primary purpose of function templates in C++?;; To enable generic programming by allowing one function definition to work with multiple data types, eliminating code duplication while maintaining type safety and performance.

What happens when you call a template function like `swap(a, b)` where `a` and `b` are integers?;; The compiler performs template instantiation, generating a specific function `swap<int>` by replacing the template parameter `T` with `int` type.

Why must template functions be defined in header files?;; Because template instantiation happens at compile time, the compiler needs to see the complete template definition when generating code for each type used.

In the template function `const T& min(const T& a, const T& b)`, what does the `const T&` return type accomplish?;; It returns a reference to avoid copying the object (important for expensive-to-copy types like strings) while preventing modification of the returned value.

What requirement must type T satisfy to work with `template<typename T> void swap(T& a, T& b)`?;; Type T must have a copy constructor and assignment operator, since the function creates a temporary copy and assigns values.

When `::min(a, b)` is called where both values are equal, which parameter is returned and why?;; The second parameter (b) is returned because the implementation uses `(a < b) ? a : b`, so when `a < b` is false (including when equal), it returns `b`.