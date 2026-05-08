---
memory: to_finish
tags:
  - to_learn
language:
  - C++
review-date: 2025-11-20
last-reviewed: 2025-10-21
scheda: done
visit-count: 3
confidence-level: 1
consecutive-correct: 0
last-struggle-date: 2025-10-21
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
# **Purpose/Why**:
---

Lambda functions solve the problem of ==creating anonymous, inline functions without the overhead of defining separate named functions or function objects. They eliminate the need for verbose function object classes when you just need a simple callable for algorithms like `std::sort`, `std::find_if`, or `std::transform`==. Lambda functions are crucial for modern C++ because they enable [[C++ Functional Programming]] paradigms, make code more readable by keeping logic close to where it's used, and provide a concise way to customize standard library algorithms. They're particularly important for event handling, callbacks, and any scenario where you need a small, one-off function that doesn't warrant a full function declaration.

# **Core Explanation:**
---

A lambda function in C++ is an <mark style="background: #BBFABBA6;">anonymous function object that can be defined inline at the point of use.</mark> Introduced in C++11, lambdas have the general syntax: ==`[capture](parameters) -> return_type { body }`.== The ==capture clause `[]` specifies which variables from the surrounding scope the lambda can access and how (by value or reference)==. ==The parameter list is optional for lambdas with no parameters. The return type can often be deduced automatically. Lambda functions are actually syntactic sugar for function objects - the compiler generates an anonymous class with an overloaded `operator()`==. They ==can capture variables from their enclosing scope, making them closures. Lambdas can be stored in variables, passed as arguments, and returned from functions, making them first-class citizens in C++.==

# **Related Concepts:**
---

- **Function Objects (Functors)**: Classes that overload `operator()` - lambdas are compiler-generated function objects. 
- **Closures**: Lambda functions that capture variables from their enclosing scope create closures. **std::function**: A wrapper that can store lambdas, function pointers, and functors in a type-erased way. 
- **Function Pointers**: Traditional C-style function pointers, but lambdas without captures can decay to function pointers. 
- **Capture Lists**: The mechanism by which lambdas access variables from their surrounding scope. 
- **Algorithm Library**: STL algorithms like `std::sort`, `std::find_if` heavily benefit from lambda usage. 
- **Template Metaprogramming**: Lambdas interact with templates and can be used in generic programming contexts.

# **Examples:**
---

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional>

int main() {
    // Basic lambda with no captures
    auto simple_lambda = []() {
        std::cout << "Hello from lambda!" << std::endl;
    };
    simple_lambda(); // Call the lambda
    
    // Lambda with parameters and return value
    auto add = [](int a, int b) -> int {
        return a + b; // Return type can be deduced, -> int is optional here
    };
    std::cout << "5 + 3 = " << add(5, 3) << std::endl;
    
    // Capture by value - lambda gets a copy of the variable
    int multiplier = 10;
    auto multiply_by_value = [multiplier](int x) {
        // multiplier is captured by value, so changes to original won't affect this
        return x * multiplier;
    };
    std::cout << "7 * 10 = " << multiply_by_value(7) << std::endl;
    
    // Capture by reference - lambda accesses the original variable
    int counter = 0;
    auto increment_counter = [&counter]() {
        counter++; // Modifies the original counter variable
    };
    increment_counter();
    std::cout << "Counter after increment: " << counter << std::endl;
    
    // Capture everything by value [=] or by reference [&]
    int base = 100;
    int offset = 25;
    auto calculate = [=](int input) { // Captures all used variables by value
        return base + offset + input;
    };
    std::cout << "Calculation: " << calculate(5) << std::endl;
    
    // Mixed capture: specific variables by reference, others by value
    int factor = 2;
    auto mixed_capture = [&counter, factor](int value) {
        counter += factor; // counter by reference, factor by value
        return value * factor;
    };
    std::cout << "Mixed result: " << mixed_capture(8) << std::endl;
    std::cout << "Counter now: " << counter << std::endl;
    
    // Using lambdas with STL algorithms
    std::vector<int> numbers = {5, 2, 8, 1, 9, 3};
    
    // Sort in descending order using lambda
    std::sort(numbers.begin(), numbers.end(), [](int a, int b) {
        return a > b; // Custom comparison: return true if a should come before b
    });
    
    std::cout << "Sorted descending: ";
    for (int n : numbers) std::cout << n << " ";
    std::cout << std::endl;
    
    // Find first number greater than 5 using lambda
    auto it = std::find_if(numbers.begin(), numbers.end(), [](int n) {
        return n > 5; // Predicate: return true for elements we're looking for
    });
    
    if (it != numbers.end()) {
        std::cout << "First number > 5: " << *it << std::endl;
    }
    
    // Transform all elements using lambda
    std::vector<int> doubled(numbers.size());
    std::transform(numbers.begin(), numbers.end(), doubled.begin(), [](int n) {
        return n * 2; // Transform each element by doubling it
    });
    
    std::cout << "Doubled numbers: ";
    for (int n : doubled) std::cout << n << " ";
    std::cout << std::endl;
    
    // Lambda stored in std::function for type erasure
    std::function<int(int, int)> operation = [](int a, int b) {
        return a * b; // Can store any callable with matching signature
    };
    std::cout << "Operation result: " << operation(4, 6) << std::endl;
    
    // Recursive lambda (C++14 and later)
    auto factorial = [](int n) {
        auto fact_impl = [](int num, auto& self) -> int {
            return num <= 1 ? 1 : num * self(num - 1, self);
        };
        return fact_impl(n, fact_impl); // Self-referencing for recursion
    };
    std::cout << "Factorial of 5: " << factorial(5) << std::endl;
    
    return 0;
}
````

# **Flashcards:**
---

What is the basic syntax of a C++ lambda function?;; `[capture](parameters) -> return_type { body }` where capture specifies variable access, parameters are optional, return type can be deduced, and body contains the function logic.

What's the difference between capture by value `[var]` and capture by reference `[&var]` in lambdas?;; Capture by value `[var]` creates a copy of the variable inside the lambda (changes don't affect original), while capture by reference `[&var]` accesses the original variable (changes affect the original).

How do you capture all variables by value vs by reference in a lambda?;; Use `[=]` to capture all used variables by value, or `[&]` to capture all used variables by reference. You can also mix: `[=, &specific_var]` captures all by value except specific_var by reference.

What type of object does the compiler generate for a lambda function?;; The compiler generates an anonymous function object (functor) class with an overloaded `operator()`. Each lambda creates a unique type, even if they have identical code.

Can a lambda without captures be converted to a function pointer?;; Yes, lambdas that don't capture any variables can be implicitly converted to function pointers, making them compatible with C-style APIs that expect function pointers.

What's the relationship between lambdas and `std::function`?;; `std::function` is a type-erased wrapper that can store lambdas, function pointers, and functors with compatible signatures. It provides a uniform interface but adds runtime overhead compared to direct lambda usage.