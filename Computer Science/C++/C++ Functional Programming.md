---
memory: to_finish
tags:
  - to_learn
language:
  - C++
review-date: 2025-11-20
last-reviewed: 2025-10-13
scheda: done
visit-count: 1
confidence-level: 1
consecutive-correct: 0
last-struggle-date: 2025-10-13
cssclasses:
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

# Purpose/Why:
---

Functional programming in C++ addresses several critical problems in modern software development:

**Primary Problems Solved:**
- **Code predictability and maintainability**: <mark style="background: #BBFABBA6;">Pure function</mark>s with no side effects <mark style="background: #BBFABBA6;">make code easier to reason about and debug</mark>
- **Concurrency safety**: Immutable data and stateless functions reduce race conditions and threading issues
- **Code reusability**: Higher-order functions and composition enable building complex operations from simple, reusable components
- **Testability**: Pure functions are inherently easier to test as they depend only on their inputs

**Importance in C++:**
<mark style="background: #ADCCFFA6;">C++ has evolved to embrace functional paradigms alongside its traditional imperative and object-oriented approaches</mark>. Since C++11, ==features like lambdas, `std::function`, and algorithms from `<algorithm>` enable developers to write more expressive, concise code. C++20 and C++23 further enhanced this with ranges, concepts, and additional functional utilities.== This paradigm is particularly valuable in:
- Algorithm-heavy applications (data processing, signal processing)
- Parallel and concurrent programming
- Generic library development
- Event-driven and reactive systems

The functional approach complements C++'s zero-overhead abstraction philosophy, allowing high-level expressiveness without sacrificing performance.

# Core Explanation:
---

**Definition:**
Functional programming in C++ is a programming paradigm that treats computation as the evaluation of mathematical functions, emphasizing immutability, pure functions, and function composition rather than mutable state and imperative control flow.

**Key Characteristics:**

1. **Pure Functions**: Functions that always produce the same output for the same input and have no side effects (don't modify external state, perform I/O, etc.)

2. **Immutability**: Data structures that cannot be modified after creation; operations return new values rather than modifying existing ones

3. **First-Class Functions**: Functions can be assigned to variables, passed as arguments, and returned from other functions

4. **Higher-Order Functions**: Functions that take other functions as parameters or return functions (`std::transform`, `std::accumulate`, etc.)

5. **Function Composition**: Combining simple functions to build more complex operations

6. **Declarative Style**: Describing *what* to compute rather than *how* to compute it step-by-step

**How It Works in C++:**

C++ implements functional programming through several language features:
- **Lambdas** (C++11+): Anonymous function objects with capture capabilities
- **`std::function`**: Type-erased wrapper for callable objects
- **Algorithms**: STL provides functional-style operations (`std::transform`, `std::filter`, `std::reduce`)
- **`constexpr`**: Compile-time function evaluation
- **Ranges** (C++20): Composable, lazy-evaluated views over sequences
- **Move semantics**: Efficiently "modify" immutable structures by transferring ownership

The paradigm works by chaining operations on data without explicit loops or mutable state, relying instead on composable transformations.

# Related Concepts:
---

1. **Lambda Expressions**
   - Anonymous functions that capture variables from their surrounding scope
   - Foundation for functional programming in modern C++
   - Enable inline function definitions without naming overhead

2. **Closures**
   - Functions that "close over" (capture) variables from their lexical scope
   - Lambdas with captures are closures in C++
   - Enable stateful function objects without explicit class definitions

3. **Functors (Function Objects)**
   - Objects that overload `operator()` to behave like functions
   - Pre-C++11 approach to creating callable objects with state
   - Lambdas are essentially compiler-generated functors

4. **Higher-Order Functions**
   - Functions that accept or return other functions
   - Examples: `std::transform`, `std::accumulate`, custom combinators
   - Enable powerful abstraction and code reuse

5. **Monads and Functional Data Structures**
   - Advanced patterns for managing effects and composition
   - `std::optional`, `std::expected` (C++23) are monadic types
   - Provide structured error handling without exceptions

6. **Ranges and Views (C++20)**
   - Composable, lazy-evaluated operations on sequences
   - Enable functional-style pipelines without intermediate copies
   - Connect functional paradigm with efficient iteration

7. **Expression Templates**
   - Technique for delaying evaluation and optimizing chains of operations
   - Used in libraries like Eigen and Blaze
   - Bridges functional style with zero-overhead performance

8. **Referential Transparency**
   - Property where expressions can be replaced with their values without changing behavior
   - Consequence of pure functions
   - Enables algebraic reasoning and compiler optimizations

**Differences from Other Paradigms:**
- **vs. Imperative**: Functional avoids explicit state mutation and loops; imperative uses mutable variables and control structures
- **vs. OOP**: Functional emphasizes data transformations; OOP emphasizes encapsulation and objects with behavior
- **Hybrid Approach**: Modern C++ encourages combining paradigms where each is most appropriate

# Examples:
---

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>
#include <functional>
#include <optional>

// ============================================================================
// EXAMPLE 1: Pure Functions vs Impure Functions
// ============================================================================

// IMPURE: Modifies external state (global variable)
int counter = 0;
int impure_increment(int x) {
    counter++;  // Side effect: modifies global state
    return x + counter;
}

// PURE: No side effects, same input always produces same output
int pure_add(int x, int y) {
    return x + y;  // Only depends on parameters, no external dependencies
}

// PURE: Immutable transformation - returns new vector instead of modifying
std::vector<int> double_values(const std::vector<int>& vec) {
    std::vector<int> result;
    result.reserve(vec.size());  // Optimization, not a side effect
    
    // Transform creates new data without modifying input
    std::transform(vec.begin(), vec.end(), std::back_inserter(result),
                   [](int x) { return x * 2; });
    return result;
}

// ============================================================================
// EXAMPLE 2: Lambda Expressions and Closures
// ============================================================================

void lambda_examples() {
    // Simple lambda: no capture, acts like a regular function
    auto add = [](int a, int b) { return a + b; };
    std::cout << "Lambda add: " << add(3, 4) << '\n';  // Output: 7
    
    // Lambda with capture by value: creates closure over 'multiplier'
    int multiplier = 10;
    auto multiply_by_captured = [multiplier](int x) { 
        return x * multiplier;  // 'multiplier' is copied into lambda
    };
    std::cout << "Captured multiply: " << multiply_by_captured(5) << '\n';  // Output: 50
    
    // Lambda with capture by reference: accesses external variable
    int sum = 0;
    auto accumulate = [&sum](int x) { 
        sum += x;  // Modifies external 'sum' - NOT pure, but useful for aggregation
    };
    accumulate(5);
    accumulate(10);
    std::cout << "Accumulated sum: " << sum << '\n';  // Output: 15
    
    // Generic lambda (C++14): works with any type
    auto generic_print = [](const auto& value) {
        std::cout << "Value: " << value << '\n';
    };
    generic_print(42);        // Works with int
    generic_print(3.14);      // Works with double
    generic_print("Hello");   // Works with const char*
}

// ============================================================================
// EXAMPLE 3: Higher-Order Functions
// ============================================================================

// Function that takes another function as parameter
template<typename Func>
int apply_twice(int value, Func func) {
    // Applies the given function two times to the value
    return func(func(value));
}

// Function that returns a function (closure)
std::function<int(int)> make_adder(int n) {
    // Returns a lambda that captures 'n' and adds it to its parameter
    return [n](int x) { return x + n; };
}

// Compose two functions: (f ∘ g)(x) = f(g(x))
template<typename F, typename G>
auto compose(F f, G g) {
    // Returns a new function that applies g first, then f
    return [f, g](auto x) { return f(g(x)); };
}

void higher_order_examples() {
    // Using apply_twice with a lambda
    auto result = apply_twice(5, [](int x) { return x * 2; });
    std::cout << "Apply twice: " << result << '\n';  // (5*2)*2 = 20
    
    // Creating and using a custom adder function
    auto add_10 = make_adder(10);
    std::cout << "Add 10 to 5: " << add_10(5) << '\n';  // Output: 15
    
    // Function composition: create pipeline of operations
    auto add_5 = [](int x) { return x + 5; };
    auto multiply_3 = [](int x) { return x * 3; };
    auto composed = compose(multiply_3, add_5);  // First add 5, then multiply by 3
    std::cout << "Composed (3*(x+5)): " << composed(10) << '\n';  // (10+5)*3 = 45
}

// ============================================================================
// EXAMPLE 4: Functional-Style Data Processing with STL Algorithms
// ============================================================================

void functional_data_processing() {
    std::vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // MAP: Transform each element (double all values)
    std::vector<int> doubled;
    doubled.reserve(numbers.size());
    std::transform(numbers.begin(), numbers.end(), std::back_inserter(doubled),
                   [](int x) { return x * 2; });
    // doubled = {2, 4, 6, 8, 10, 12, 14, 16, 18, 20}
    
    // FILTER: Keep only elements matching predicate (keep even numbers)
    std::vector<int> evens;
    std::copy_if(numbers.begin(), numbers.end(), std::back_inserter(evens),
                 [](int x) { return x % 2 == 0; });
    // evens = {2, 4, 6, 8, 10}
    
    // REDUCE: Combine all elements into single value (sum all numbers)
    int sum = std::accumulate(numbers.begin(), numbers.end(), 0);
    // sum = 1+2+3+4+5+6+7+8+9+10 = 55
    
    // REDUCE with custom operation (product of all numbers)
    int product = std::accumulate(numbers.begin(), numbers.end(), 1,
                                  [](int acc, int x) { return acc * x; });
    // product = 1*2*3*4*5*6*7*8*9*10 = 3628800
    
    // CHAINING OPERATIONS: Filter evens, double them, sum the result
    std::vector<int> filtered_evens;
    std::copy_if(numbers.begin(), numbers.end(), std::back_inserter(filtered_evens),
                 [](int x) { return x % 2 == 0; });
    
    std::vector<int> doubled_evens;
    std::transform(filtered_evens.begin(), filtered_evens.end(), 
                   std::back_inserter(doubled_evens),
                   [](int x) { return x * 2; });
    
    int final_sum = std::accumulate(doubled_evens.begin(), doubled_evens.end(), 0);
    std::cout << "Chained result: " << final_sum << '\n';  // (2+4+6+8+10)*2 = 60
}

// ============================================================================
// EXAMPLE 5: Immutability and Const-Correctness
// ============================================================================

// Demonstrates functional style with immutability
class ImmutablePoint {
    const int x_;  // Members are const - cannot be modified after construction
    const int y_;
    
public:
    ImmutablePoint(int x, int y) : x_(x), y_(y) {}
    
    // Getters are const - don't modify object state
    int x() const { return x_; }
    int y() const { return y_; }
    
    // Instead of modifying, return NEW object with updated values
    ImmutablePoint move(int dx, int dy) const {
        return ImmutablePoint(x_ + dx, y_ + dy);  // Creates new instance
    }
    
    // Pure function: always produces same result for same input
    int distance_from_origin() const {
        return x_ * x_ + y_ * y_;  // Simplified distance (squared)
    }
};

void immutability_example() {
    const ImmutablePoint p1(3, 4);  // Original point
    const ImmutablePoint p2 = p1.move(1, 1);  // New point, p1 unchanged
    
    std::cout << "p1: (" << p1.x() << ", " << p1.y() << ")\n";  // (3, 4)
    std::cout << "p2: (" << p2.x() << ", " << p2.y() << ")\n";  // (4, 5)
    // p1 remains unchanged - immutability guarantees no side effects
}

// ============================================================================
// EXAMPLE 6: Functional Error Handling with std::optional
// ============================================================================

// Returns optional value: Some(result) or None if division by zero
std::optional<double> safe_divide(double numerator, double denominator) {
    if (denominator == 0.0) {
        return std::nullopt;  // Represents absence of value (error case)
    }
    return numerator / denominator;  // Represents successful computation
}

// Monadic composition: chain operations that might fail
std::optional<double> compute_complex(double a, double b, double c) {
    // Each operation might fail, but we handle it functionally
    auto step1 = safe_divide(a, b);
    if (!step1) return std::nullopt;  // Early return if first step failed
    
    auto step2 = safe_divide(*step1, c);
    return step2;  // Propagate result or failure
}

void optional_example() {
    // Success case: all operations valid
    auto result1 = compute_complex(100.0, 5.0, 4.0);
    if (result1) {
        std::cout << "Result: " << *result1 << '\n';  // (100/5)/4 = 5.0
    }
    
    // Failure case: division by zero handled gracefully
    auto result2 = compute_complex(100.0, 0.0, 4.0);
    if (!result2) {
        std::cout << "Computation failed\n";  // No exception, just None
    }
    
    // Using functional transform with optional
    auto value = safe_divide(10.0, 2.0);
    auto transformed = value.transform([](double x) { return x * 2; });
    // transformed contains 10.0 (since (10/2)*2 = 10)
}

// ============================================================================
// MAIN: Demonstrating all examples
// ============================================================================

int main() {
    std::cout << "=== Lambda Examples ===\n";
    lambda_examples();
    
    std::cout << "\n=== Higher-Order Functions ===\n";
    higher_order_examples();
    
    std::cout << "\n=== Functional Data Processing ===\n";
    functional_data_processing();
    
    std::cout << "\n=== Immutability Example ===\n";
    immutability_example();
    
    std::cout << "\n=== Optional Example ===\n";
    optional_example();
    
    return 0;
}
````

# Flashcards:

---

What is a pure function in C++ functional programming?;; A pure function is a function that always produces the same output for the same input and has no side effects (doesn't modify external state or perform I/O). Example: `int add(int a, int b) { return a + b; }` is pure, while a function that modifies global variables or prints to console is impure.

What are higher-order functions and provide two STL examples?;; Higher-order functions are functions that take other functions as parameters or return functions as results. STL examples include `std::transform` (applies a function to each element in a range) and `std::accumulate` (reduces a range using a binary function). They enable code reusability and composition.

Explain lambda captures in C++ - what is the difference between `[x]` and `[&x]`?;; `[x]` captures variable x by value (copies it into the lambda), making the lambda independent of the original variable. `[&x]` captures by reference, allowing the lambda to access and potentially modify the original variable. Capture by value is more aligned with pure functional programming, while capture by reference enables stateful operations.

What is immutability and how does it benefit concurrent programming in C++?;; Immutability means data cannot be modified after creation; operations return new values instead of modifying existing ones. In concurrent programming, immutable data eliminates race conditions since multiple threads can safely read shared data without locks. No thread can change data another thread is reading, preventing data races.

How does `std::optional` enable functional-style error handling?;; `std::optional<T>` represents a value that may or may not exist, similar to Maybe/Option types in functional languages. It enables error handling without exceptions by returning `std::nullopt` for errors or a valid value for success. Functions can be chained using `transform` or checked with `has_value()`, making error propagation explicit and composable.

What is function composition and how is it implemented in C++?;; Function composition combines simple functions to create more complex operations, mathematically written as (f ∘ g)(x) = f(g(x)). In C++, it's implemented using lambdas that capture functions: `auto compose = [f, g](auto x) { return f(g(x)); }`. This enables building pipelines of transformations in a declarative, reusable way.