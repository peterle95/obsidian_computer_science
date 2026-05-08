---
memory: to_finish
tags:
 - learned
language:
 - C++
review-date:
last-reviewed: 2025-08-20
scheda: done
visit-count: 4
confidence-level: 3
consecutive-correct: 4
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
Templates in C++ solve the fundamental problem of code duplication and type-specific programming by enabling generic programming. They allow developers to write code that works with ==multiple data types without rewriting the same logic for each type==. This is crucial because it promotes code reusability, type safety, and maintainability while avoiding the performance overhead of runtime polymorphism. Templates enable the creation of flexible, efficient libraries (like the STL) and allow compile-time code generation, making C++ both powerful and performant. They're essential for creating scalable software architectures where algorithms and data structures can work with various types while maintaining compile-time type checking.

The fundamental problems templates solve:

- **Type-specific code duplication**: ==Instead of writing `swapInt`, `swapFloat`, `swapString`, you write one `swap<T>` template==
- **Compile-time type safety**: <mark style="background:

# FF5582A6;">Templates are resolved at compile time</mark>, ensuring type safety without runtime overhead
- **Performance**: No runtime polymorphism overhead like virtual functions - templates generate optimized code for each type used
- **Code maintainability**: One template definition means one place to fix bugs or add features

Templates are crucial in C++ because they enable the Standard Template Library (STL), generic algorithms, and modern C++ design patterns while maintaining C++'s zero-overhead principle.

# **Core Explanation:**

---
Templates in C++ are a feature that allows you to write generic code that can work with different data types. ==They enable compile-time polymorphism== by allowing the compiler to generate specific versions of functions or classes for different types.

Key characteristics include:
- **Compile-time Code Generation**: Templates are expanded at compile time, creating specific instances for each type used
- **Type Safety**: Full type checking is performed at compile time, preventing type-related errors
- **Zero Runtime Overhead**: No performance penalty compared to hand-written type-specific code
- **Template Parameters**: Can accept type parameters, non-type parameters (values), and template template parameters
- **Specialization**: Ability to provide custom implementations for specific types
- **[[C++ - Template Argument Deduction]]**: Compiler can often infer template parameters automatically

<mark style="background:

# FFB8EBA6;">Templates work through a process called template instantiation, where the compiler generates actual code by substituting template parameters with concrete types.</mark> This happens during compilation, not runtime, which means each unique combination of template arguments creates a separate copy of the code optimized for those specific types.

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
- **[[C++ - Templates vs Function Overloading]]
- **Generic Programming**: Programming paradigm enabled by templates, focusing on writing algorithms independent of specific data types
- **Template Specialization**: Providing custom implementations for specific types or values, allowing fine-tuned behavior
- **[[C++ - Template Metaprogramming]]**: Using templates to perform computations at compile time, creating programs that generate other programs
- **SFINAE (Substitution Failure Is Not An Error)**: Template technique that allows overload resolution based on template parameter validity
- **[[C++ - Concepts]]**: Constraints on template parameters that make template code more readable and provide better error messages
- **Function Overloading**: Related mechanism for providing multiple implementations, but templates provide a more general solution
- **Inheritance vs Templates**: Both provide polymorphism, but templates provide compile-time polymorphism while inheritance provides runtime polymorphism
- **STL (Standard Template Library)**: Extensive use of templates in containers, algorithms, and iterators
- **[[C++ - Template Header-Only Implementation]]** - Why template code must be in header files (unlike regular C++ code). This is essential for actually using templates in practice.

# **Examples:**

---
```cpp

# include <iostream>

# include <vector>

# include <string>

# include <type_traits>

// 1. FUNCTION TEMPLATES
// Basic function template - works with any type that supports comparison
template<typename T>
T maximum(T a, T b) {
 // Template parameter T will be replaced with actual type during compilation
 return (a > b) ? a : b;
}

// Function template with multiple parameters
template<typename T, typename U>
auto add(T a, U b) -> decltype(a + b) {
 // Auto return type deduction based on the operation result
 return a + b;
}

// Function template with non-type parameter
template<typename T, int Size>
void print_array(T (&arr)[Size]) {
 // Size is a compile-time constant, not a runtime variable
 std::cout << "Array of size " << Size << ": ";
 for (int i = 0; i < Size; ++i) {
 std::cout << arr[i] << " ";
 }
 std::cout << std::endl;
}

// 2. CLASS TEMPLATES
// Basic class template - generic container
template<typename T>
class SimpleVector {
private:
 T* data; // Pointer to dynamically allocated array of type T
 size_t size; // Current number of elements
 size_t capacity; // Maximum capacity before reallocation needed

public:
 // Constructor - initializes empty vector
 SimpleVector : data(nullptr), size(0), capacity(0) {}

 // Constructor with initial capacity
 explicit SimpleVector(size_t initial_capacity)
 : data(new T[initial_capacity]), size(0), capacity(initial_capacity) {}

 // Destructor - cleans up dynamically allocated memory
 ~SimpleVector {
 delete data;
 }

 // Copy constructor - deep copy for template class
 SimpleVector(const SimpleVector& other)
 : data(new T[other.capacity]), size(other.size), capacity(other.capacity) {
 for (size_t i = 0; i < size; ++i) {
 data[i] = other.data[i];
 }
 }

 // Add element to vector
 void push_back(const T& element) {
 if (size >= capacity) {
 // Need to reallocate - double the capacity
 size_t new_capacity = (capacity == 0) ? 1 : capacity * 2;
 T* new_data = new T[new_capacity];

 // Copy existing elements to new memory
 for (size_t i = 0; i < size; ++i) {
 new_data[i] = data[i];
 }

 delete data;
 data = new_data;
 capacity = new_capacity;
 }
 data[size++] = element;
 }

 // Access element by index
 T& operator {
 return data[index];
 }

 // Get current size
 size_t get_size const {
 return size;
 }

 // Display all elements
 void display const {
 std::cout << "Vector contents: ";
 for (size_t i = 0; i < size; ++i) {
 std::cout << data[i] << " ";
 }
 std::cout << std::endl;
 }
};

// 3. TEMPLATE SPECIALIZATION
// Generic template for converting to string
template<typename T>
std::string to_string_custom(const T& value) {
 return "Generic conversion not implemented";
}

// Specialized template for int
template<>
std::string to_string_custom<int>(const int& value) {
 return std::to_string(value);
}

// Specialized template for double
template<>
std::string to_string_custom<double>(const double& value) {
 return std::to_string(value);
}

// 4. TEMPLATE WITH MULTIPLE PARAMETERS AND DEFAULT VALUES
template<typename T, typename Compare = std::less<T>>
class PriorityQueue {
private:
 std::vector<T> heap;
 Compare comp; // Comparison function object

 // Helper function to maintain heap property
 void heapify_up(size_t index) {
 while (index > 0) {
 size_t parent = (index - 1) / 2;
 if (comp(heap[index], heap[parent])) {
 std::swap(heap[index], heap[parent]);
 index = parent;
 } else {
 break;
 }
 }
 }

public:
 // Constructor with optional comparison function
 PriorityQueue(Compare c = Compare) : comp(c) {}

 // Insert element
 void push(const T& element) {
 heap.push_back(element);
 heapify_up(heap.size - 1);
 }

 // Check if queue is empty
 bool empty const {
 return heap.empty;
 }

 // Get top element
 const T& top const {
 return heap;
 }

 // Remove top element
 void pop {
 if (!empty) {
 heap = heap.back;
 heap.pop_back;
 // Would need heapify_down implementation for complete functionality
 }
 }
};

// 5. VARIADIC TEMPLATES (C++11 and later)
// Template that accepts variable number of arguments
template<typename T>
void print_all(const T& value) {
 // Base case - single argument
 std::cout << value << std::endl;
}

template<typename T, typename... Args>
void print_all(const T& first, const Args&... rest) {
 // Recursive case - print first, then recursively print the rest
 std::cout << first << " ";
 print_all(rest...); // Recursively call with remaining arguments
}

// 6. TEMPLATE METAPROGRAMMING EXAMPLE
// Compile-time factorial calculation
template<int N>
struct Factorial {
 static constexpr int value = N * Factorial<N-1>::value;
};

// Specialization for base case
template<>
struct Factorial<0> {
 static constexpr int value = 1;
};

// Demonstration of all template concepts
int main {
 std::cout << "=== FUNCTION TEMPLATES ===" << std::endl;

 // Template argument deduction - compiler infers types
 std::cout << "Max of 5 and 3: " << maximum(5, 3) << std::endl;
 std::cout << "Max of 3 and 2.71: " << maximum(3.14, 2.71) << std::endl;
 std::cout << "Max of 'a' and 'z': " << maximum('a', 'z') << std::endl;

 // Mixed type template
 std::cout << "Add 5 + 3.14: " << add(5, 3.14) << std::endl;

 // Non-type template parameter
 int arr = {1, 2, 3, 4, 5};
 print_array(arr); // Size automatically deduced

 std::cout << "\n=== CLASS TEMPLATES ===" << std::endl;

 // Template class instantiation for different types
 SimpleVector<int> int_vector;
 SimpleVector<std::string> string_vector;

 // Using int vector
 int_vector.push_back(10);
 int_vector.push_back(20);
 int_vector.push_back(30);
 int_vector.display;

 // Using string vector
 string_vector.push_back("Hello");
 string_vector.push_back("World");
 string_vector.push_back("Templates");
 string_vector.display;

 std::cout << "\n=== TEMPLATE SPECIALIZATION ===" << std::endl;

 // Using specialized templates
 std::cout << "Int to string: " << to_string_custom(42) << std::endl;
 std::cout << "Double to string: " << to_string_custom(3.14159) << std::endl;
 std::cout << "Char to string: " << to_string_custom('A') << std::endl;

 std::cout << "\n=== TEMPLATE WITH DEFAULT PARAMETERS ===" << std::endl;

 // Priority queue with default comparison (min-heap)
 PriorityQueue<int> min_heap;
 min_heap.push(5);
 min_heap.push(1);
 min_heap.push(3);
 std::cout << "Min heap top: " << min_heap.top << std::endl;

 // Priority queue with custom comparison (max-heap)
 PriorityQueue<int, std::greater<int>> max_heap;
 max_heap.push(5);
 max_heap.push(1);
 max_heap.push(3);
 std::cout << "Max heap top: " << max_heap.top << std::endl;

 std::cout << "\n=== VARIADIC TEMPLATES ===" << std::endl;

 // Variable number of arguments with different types
 print_all(1, 2.5, "Hello", 'X', true);

 std::cout << "\n=== TEMPLATE METAPROGRAMMING ===" << std::endl;

 // Compile-time computation
 std::cout << "Factorial of 5: " << Factorial<5>::value << std::endl;
 std::cout << "Factorial of 10: " << Factorial<10>::value << std::endl;

 return 0;
}
````

# **Flashcards:**

---
What are C++ templates and when are they processed?;; Templates are a feature that allows writing generic code that works with different data types. They are processed at compile time, generating specific code instances for each type used.

What is the difference between function templates and class templates?;; Function templates create generic functions that work with different parameter types, while class templates create generic classes that can store and manipulate different data types.

What is template specialization and why is it useful?;; Template specialization allows providing custom implementations for specific types or values, enabling optimized or different behavior for particular cases while maintaining the generic template for other types.

What are the main advantages of using templates over runtime polymorphism?;; Templates provide compile-time polymorphism with zero runtime overhead, full type safety, and better performance since the compiler generates optimized code for each specific type combination.

What are variadic templates and what problem do they solve?;; Variadic templates accept a variable number of template arguments, allowing functions and classes to work with any number of parameters of different types, solving the problem of having to write multiple overloads.

What is template metaprogramming?;; Template metaprogramming is the practice of using templates to perform computations and generate code at compile time, allowing for optimizations and calculations that happen during compilation rather than runtime.