---
memory: to_finish
tags:
 - learned
language:
 - C++
review-date:
last-reviewed: 2025-09-02
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
The global scope resolution operator (`::`) solves the fundamental ==problem of **name ambiguity** in C++==. When ==multiple functions, variables, or types have the same name in different scopes (global, class, namespace), the compiler needs a way to distinguish which one you're referring to==. This operator provides explicit control over scope resolution, preventing naming conflicts and ensuring you access the intended identifier.

This is crucial in C++ because:
- **Large codebases** often have naming collisions
- **Object-oriented programming** creates multiple scopes (classes, namespaces)
- **Template instantiation** can create ambiguous contexts
- **Library integration** may introduce conflicting names

# **Core Explanation:**

---
The **global scope resolution operator** `::` is a unary operator that explicitly specifies the global namespace when placed at the beginning of an identifier. It forces the compiler to look for the identifier in the global scope, bypassing any local or class-level names that might shadow it.

**Key Characteristics:**
- **Syntax**: `::identifier` (when used globally)
- **Precedence**: Highest precedence (left-to-right associativity)
- **Scope bypass**: Ignores local/member scope, goes directly to global
- **Compile-time resolution**: Resolved during compilation, not runtime
- **Works with**: Functions, variables, types, templates

**How it works:**
1. Compiler encounters `::identifier`
2. Searches only in global namespace
3. Ignores any local or member identifiers with same name
4. Resolves to global definition or compilation error if not found

The operator can also be used for namespace resolution (`namespace::identifier`) but when used alone (`::identifier`), it specifically targets the global scope.

# **Related Concepts:**

---
**[[C++ - Namespaces]]**: Logical containers for identifiers. `::` can navigate between namespaces (`std::cout`) or access global scope from within namespaces.

**Name Hiding/Shadowing**: When local variables hide global ones with same name. `::` provides access to hidden global identifiers.

**Scope Resolution Operator `::`**: The broader concept - `::` can resolve any scope (class members, namespaces). Global scope is just one application.

**Function Overloading**: Multiple functions with same name but different parameters. `::` helps distinguish between global and member versions.

**Template Instantiation**: Templates create functions/classes at compile-time. `::` ensures you call the intended template specialization.

**Using Directives**: `using namespace std;` brings namespace contents into current scope. `::` can bypass these to access global scope directly.

**ADL (Argument-Dependent Lookup)**: Compiler automatically looks in namespaces of function arguments. `::` can override this behavior.

# **Examples:**

---
```cpp

# include <iostream>

# include "iter.hpp"

// Global template function defined in global namespace
template<typename T>
void print(const T& value) {
 std::cout << value << " ";
}

// Global template function - this is what :: refers to
template<typename T, typename F>
void iter(T* array, std::size_t length, F func) {
 if (!array) return;
 for (std::size_t i = 0; i < length; ++i) {
 func(array[i]);
 }
}

class ArrayProcessor {
private:
 // Member function with same name as global template
 // This creates potential naming conflict
 template<typename T>
 void iter(T* array, std::size_t length) {
 std::cout << "Member iter called\n";
 // This member version doesn't take a function parameter
 }

public:
 void processArray {
 int numbers = {1, 2, 3, 4, 5};
 std::size_t size = sizeof(numbers) / sizeof(numbers);

 // WITHOUT :: - would call member function if it matched signature
 // This would cause compilation error because member iter
 // doesn't accept third parameter
 // iter(numbers, size, print<int>); // ERROR!

 // WITH :: - explicitly calls global template function
 // :: tells compiler: "ignore any member functions, use global scope"
 ::iter(numbers, size, print<int>);

 // This works because :: bypasses member scope entirely
 std::cout << std::endl;
 }

 void demonstrateConflict {
 int arr = {1, 2, 3};
 std::size_t size = 3;

 // Calls member function (no :: prefix)
 // Compiler finds this in current class scope first
 iter(arr, size);

 // Calls global function (with :: prefix)
 // :: forces compiler to look in global scope only
 ::iter(arr, size, print<int>);
 std::cout << std::endl;
 }
};

// Example from the exercise showing defensive programming
void demonstrateExerciseUsage {
 int numbers = {1, 2, 3, 4, 5};
 std::size_t numbersSize = sizeof(numbers) / sizeof(numbers);

 // From exercise: :: used for explicit global scope access
 // Even though no conflicts exist here, :: makes intent clear
 std::cout << "Exercise style - explicit global scope: ";
 ::iter(numbers, numbersSize, print<int>);
 std::cout << std::endl;

 // This would work identically (no conflicts in this context)
 // But :: makes code more explicit and defensive
 std::cout << "Without :: (works the same here): ";
 iter(numbers, numbersSize, print<int>);
 std::cout << std::endl;
}

// Namespace example showing different uses of ::
namespace MyNamespace {
 template<typename T>
 void print(const T& value) {
 std::cout << "[NS]" << value << " ";
 }

 void testScopeResolution {
 int arr = {10, 20, 30};
 std::size_t size = 3;

 // Calls function from MyNamespace (current namespace)
 ::iter(arr, size, print<int>);
 std::cout << std::endl;

 // Calls global print function (bypasses namespace)
 ::iter(arr, size, ::print<int>);
 std::cout << std::endl;
 }
}

int main {
 std::cout << "=== Global Scope Resolution Operator Demo ===" << std::endl;

 // Demonstrate exercise usage
 demonstrateExerciseUsage;

 // Demonstrate class member conflicts
 ArrayProcessor processor;
 processor.processArray;
 processor.demonstrateConflict;

 // Demonstrate namespace usage
 MyNamespace::testScopeResolution;

 return 0;
}

/*
KEY POINTS DEMONSTRATED:
1. :: forces global scope lookup, ignoring local/member scope
2. Without ::, compiler uses normal scope resolution rules
3. :: is defensive programming - makes intent explicit
4. Particularly useful when names conflict across scopes
5. Works with templates, functions, variables, and types
6. Resolved at compile-time, no runtime overhead
*/
````

# **Flashcards:**

---
What does the :: operator do when placed before an identifier?;; Forces the compiler to look for that identifier only in the global namespace, bypassing any local or member scope identifiers with the same name

Why would you use ::iter instead of just iter in the exercise code?;; To explicitly specify you want the global template function, not any potential member function with the same name. It's defensive programming that makes intent clear and prevents naming conflicts

In what scenarios is the global scope resolution operator most useful?;; When there are naming conflicts between global functions and class members, when working in namespaces that might shadow global names, and in large codebases where explicit scope specification improves code clarity

What happens if you use :: with an identifier that doesn't exist in global scope?;; The compiler will generate an error because it only searches the global namespace and cannot find the specified identifier

Can the :: operator be used with template functions?;; Yes, :: works with templates just like regular functions. For example, ::iter<int> explicitly calls the global template function instantiated for int type

What's the difference between ::identifier and namespace::identifier?;; ::identifier specifically targets the global scope, while namespace::identifier targets a named namespace. The global scope version bypasses all other scopes entirely