---
memory: to_finish
tags:
  - mastered
language:
  - C++
review-date: ""
last-reviewed: 2025-10-16
scheda: done
visit-count: 4
confidence-level: 3
consecutive-correct: 4
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

The `override` keyword in C++ primarily solves the problem of ensuring that a member function in a derived class is indeed overriding a virtual function in its base class. Without `override`, subtle mistakes in function signature (e.g., typos in name, incorrect parameter types, or differing `const` qualification) can lead to a new, unrelated function being defined in the derived class instead of the intended override. This results in unintended behavior, especially in polymorphic scenarios where a base class pointer or reference is used to call the function.

It is important in C++ because it significantly improves code clarity and safety, turning potential runtime errors (due to incorrect function calls) into compile-time errors. This allows developers to catch mistakes early in the development cycle, making the code more robust and easier to maintain, especially in complex class hierarchies. It directly supports and strengthens the concept of polymorphism by enforcing correct method overriding.

# **Core Explanation:**
---

The `override` keyword is a ==_contextual keyword_ (meaning it only has special meaning in certain contexts) in C++11 and later, used in the declaration of member functions in derived classes.== Its purpose is to explicitly state that the function is intended to <mark style="background: #FF5582A6;">override a virtual function in a base class.
</mark>
Key characteristics and how it works:

- **Compiler Enforcement:** When `override` is used, the compiler checks if the declared function actually overrides a virtual function from a base class with the _exact same signature_ (including name, parameter types, `const`/`volatile` qualifiers, and return type).
- **Compile-Time Error:** If the compiler determines that the function marked `override` does _not_ correctly override a base class virtual function (e.g., due to a mismatch in name, parameter list, or return type), it will issue a compile-time error. This prevents silent bugs that would otherwise occur if a new function was accidentally defined instead of an override.
- **Clarity and Intent:** It serves as clear documentation for other developers (and your future self) that this function is meant to be an override, improving readability and understanding of the class hierarchy.
- **Virtual Functions Only:** `override` can _only_ be used with functions that are declared `virtual` in a base class. You cannot use `override` on a non-virtual function, nor on a function that doesn't exist in a base class's virtual function table.
- **Not Mandatory, But Recommended:** While `override` is not strictly mandatory for a function to override a virtual base class function, its use is highly recommended as a best practice for all overriding functions due to the safety and clarity it provides.

In essence, `override` acts as a safety net that catches common mistakes in polymorphism by enforcing strict signature matching at compile time.
# **Related Concepts:**
---

- **Virtual Functions:** The `override` keyword is inextricably linked to virtual functions. A function in a derived class can only `override` a function that was declared `virtual` in its base class. Virtual functions enable polymorphism (calling the correct derived class function through a base class pointer/reference), and `override` ensures that this polymorphic behavior is correctly set up.
- **Polymorphism:** The ability of an object to take on many forms. In C++, this often refers to runtime polymorphism achieved through virtual functions and inheritance. `override` is a tool to ensure that the functions contributing to this polymorphism are correctly implemented in derived classes. Without correct overriding (which `override` helps enforce), runtime polymorphism wouldn't work as expected for those functions.
- **Inheritance:** The mechanism by which one class acquires the properties and behaviors of another class. `override` is used exclusively within an inheritance hierarchy where a derived class is specializing or modifying the behavior of a base class's virtual function.
- **Function Overloading:** Defining multiple functions in the same scope with the same name but different parameter lists. This is a form of compile-time polymorphism. `override` is distinct from overloading ([[C++ - overloading vs overriding]]); `override` deals with functions with the _exact same signature_ in different classes within an inheritance hierarchy, while overloading deals with functions with _different signatures_ in the same scope.
- **Function Hiding (Name Hiding):** When a derived class declares a function with the same name as a base class function (virtual or non-virtual) but with a different signature, it "hides" the base class function. If this happens accidentally when you intend to override a virtual function, it leads to subtle bugs because the base class function might be called unexpectedly. `override` explicitly prevents this by flagging such scenarios as errors if the intent was to override.
# **Examples:**
---
```js
#include <iostream>
#include <string>

// Base class with virtual functions
class Base {
public:
    virtual void greet() const {
        std::cout << "Base says hello!" << std::endl;
    }

    virtual void calculate(int a, int b) {
        std::cout << "Base calculates: " << (a + b) << std::endl;
    }

    // A virtual function intended to be overridden, but with a potential for typo.
    virtual void doSomethingImportant() {
        std::cout << "Base doing something important." << std::endl;
    }

    // A non-virtual function
    void nonVirtualFunction() {
        std::cout << "Base non-virtual function." << std::endl;
    }

    virtual ~Base() = default; // Virtual destructor is good practice for polymorphic classes
};

// Derived class demonstrating correct override
class DerivedA : public Base {
public:
    // Correctly overrides greet() from Base.
    // The 'override' keyword clearly states this intent and the compiler verifies it.
    void greet() const override {
        std::cout << "DerivedA says greetings!" << std::endl;
    }

    // Correctly overrides calculate() from Base.
    void calculate(int x, int y) override { // Parameters names can differ, but types must match
        std::cout << "DerivedA calculates: " << (x * y) << std::endl;
    }

    // Correctly overrides doSomethingImportant() from Base.
    void doSomethingImportant() override {
        std::cout << "DerivedA doing something even more important!" << std::endl;
    }

    // Attempting to override a non-virtual function with 'override' will cause a compile error.
    // void nonVirtualFunction() override { // ERROR: 'nonVirtualFunction' does not override any base class methods
    //     std::cout << "DerivedA trying to override non-virtual." << std::endl;
    // }
};

// Derived class demonstrating potential issues without 'override'
class DerivedB : public Base {
public:
    // This is a common mistake: A typo in the function name.
    // Without 'override', the compiler sees this as a *new* function in DerivedB,
    // not an override of Base::greet(). This leads to subtle bugs in polymorphism.
    void greett() const { // Typo: 'greett' instead of 'greet'
        std::cout << "DerivedB says hello (typo version)!" << std::endl;
    }

    // Another common mistake: Mismatched parameter types.
    // Without 'override', this is a *new* function, not an override.
    void calculate(double a, double b) { // Parameter types mismatch (double vs int)
        std::cout << "DerivedB calculates with doubles: " << (a / b) << std::endl;
    }

    // Mismatched const-qualification.
    // Without 'override', this is a *new* function, not an override.
    // virtual void doSomethingImportant() { // Missing 'const' or different signature from base
    //     std::cout << "DerivedB doing something important (no const)." << std::endl;
    // }
};

// Derived class demonstrating how 'override' catches errors
class DerivedC : public Base {
public:
    // Using 'override' to catch the typo:
    // This will produce a COMPILE-TIME ERROR because 'greett' does not override Base::greet().
    // void greett() const override { // ERROR: 'greett' does not override any base class methods
    //     std::cout << "DerivedC (with override) says hello (typo version)!" << std::endl;
    // }

    // Using 'override' to catch parameter type mismatch:
    // This will produce a COMPILE-TIME ERROR because calculate(double, double)
    // does not override Base::calculate(int, int).
    // void calculate(double a, double b) override { // ERROR: 'calculate' does not override any base class methods
    //     std::cout << "DerivedC (with override) calculates with doubles." << std::endl;
    // }

    // Correct override
    void greet() const override {
        std::cout << "DerivedC says hello correctly!" << std::endl;
    }
};

int main() {
    std::cout << "--- Base Class Object ---" << std::endl;
    Base b_obj;
    b_obj.greet();
    b_obj.calculate(5, 3);
    b_obj.doSomethingImportant();
    std::cout << std::endl;

    std::cout << "--- DerivedA Object (Correct Overrides with 'override') ---" << std::endl;
    DerivedA da_obj;
    da_obj.greet();          // Calls DerivedA::greet()
    da_obj.calculate(5, 3);  // Calls DerivedA::calculate()
    da_obj.doSomethingImportant(); // Calls DerivedA::doSomethingImportant()
    std::cout << std::endl;

    std::cout << "--- DerivedB Object (Potential issues without 'override') ---" << std::endl;
    DerivedB db_obj;
    db_obj.greett();         // Calls DerivedB::greett() (the new function)
    db_obj.calculate(5.0, 2.0); // Calls DerivedB::calculate(double, double) (the new function)
    // If you intend to call the base class calculate with ints, you'd have issues here.
    std::cout << std::endl;

    std::cout << "--- Polymorphic Calls ---" << std::endl;

    // Base class pointer pointing to DerivedA object
    Base* ptrA = &da_obj;
    std::cout << "ptrA->greet(): ";
    ptrA->greet(); // Calls DerivedA::greet() because it's a virtual function and correctly overridden.
    std::cout << "ptrA->calculate(5, 3): ";
    ptrA->calculate(5, 3); // Calls DerivedA::calculate() because it's virtual and correctly overridden.
    std::cout << "ptrA->doSomethingImportant(): ";
    ptrA->doSomethingImportant(); // Calls DerivedA::doSomethingImportant()
    std::cout << std::endl;

    // Base class pointer pointing to DerivedB object
    Base* ptrB = &db_obj;
    std::cout << "ptrB->greet(): ";
    // This calls Base::greet() NOT DerivedB::greett()!
    // This is the subtle bug 'override' helps prevent. DerivedB::greett() didn't override anything.
    ptrB->greet();
    std::cout << "ptrB->calculate(5, 3): ";
    // This calls Base::calculate() NOT DerivedB::calculate(double, double)!
    // Again, 'override' would catch this mismatch.
    ptrB->calculate(5, 3);
    ptrB->doSomethingImportant(); // Calls Base::doSomethingImportant() because DerivedB's 'doSomethingImportant' (if it existed) didn't correctly override.
    std::cout << std::endl;

    // Now, let's see how DerivedC with 'override' catches errors at compile time.
    // If you uncomment the erroneous 'greett()' or 'calculate(double, double)' in DerivedC,
    // the code will fail to compile, preventing the runtime issues seen with DerivedB.
    std::cout << "--- DerivedC Object (Compile-time error catching with 'override') ---" << std::endl;
    DerivedC dc_obj;
    dc_obj.greet(); // This correctly calls the overridden function.
    std::cout << std::endl;

    return 0;
}
```
# **Flashcards:**
---
What is the purpose of the `override` keyword in C++?;; To explicitly state that a member function in a derived class is intended to override a virtual function in a base class, and to enable the compiler to check this intent. 

What happens if a function is marked `override` but does not correctly override a base class virtual function?;; The compiler will issue a compile-time error, preventing potential runtime bugs due to unintended function hiding or new function definition. 

Can `override` be used with non-virtual functions?;; No, `override` can only be used with functions that are declared `virtual` in a base class.